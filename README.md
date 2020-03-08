---
description: 2020 2 29 스터디 내용
---

# reserved\_mem 분석 노트

### 1. Before you read

    **Have to read** [reserved-memory.txt ](https://www.kernel.org/doc/Documentation/devicetree/bindings/reserved-memory/reserved-memory.txt)carefully.



#### 1.1 reserved-memory **간단 정**

    **Reserved-memory 노드**에 기술된 프로퍼티는 다음과 같다.

| Reserved-memory 노드의 property | value |
| :--- | :--- |
| **\#address-cells, \#size-cells \(필수 속성\)** | 루트 노드의 값과 동일해야 한다. |
| **ranges \(필수 속성\)** | 값은 비어 있어야 한다.\(empty\) |

    **Reserved-memory의 자식 노드**들은 1개 이상의 예약 메모리 영역을 가진다. 예약 메모리 영역은 

1. **reg 프로퍼티**를 사용하여 예약된 메모리의 특정 범위\(base, size\)를 지정하거나,
2. **size 프로퍼티**를 사용하여 동적으로 예약받을 수 있다. \(이 옵션에 더불어 **alignment, alloc-ranges 프로퍼티**를 같이 사용할 수 있다.\)

{% hint style="warning" %}
**자식 노드는 반드시 둘 중 하나\(reg, size\)의 프로퍼티는 가져야 한다.**
{% endhint %}

{% hint style="danger" %}
**나머지 프로퍼티는 문서를 참고하라.**
{% endhint %}

### 2. Function schematic

```text
┗━start_kernel
  ┗━setup_arch
    ┗━arm64_memblock_init
      ┗━early_init_fdt_scan_reserved_mem  
        ┠━of_scan_flat_dt                 <----------------------------- START
        ┃ ┗━__fdt_scan_reserved_mem       
        ┃   ┠━____reserved_mem_reserve_reg
        ┠   ┗━fdt_reserved_mem_save_node
        ┗━fdt_init_reserved_mem
          ┠━__rmem_check_for_overlap
          ┠━__reserved_mem_alloc_size
          ┗━__reserved_mem_init_node
```

### 3. Dive into function

{% tabs %}
{% tab title="\_\_fdt\_scan\_reserved\_mem" %}
```c
static int __init __fdt_scan_reserved_mem(unsigned long node, const char *uname,
                      int depth, void *data)
{
    static int found;
    int err;

    if (!found && depth == 1 && strcmp(uname, "reserved-memory") == 0) {
        if (__reserved_mem_check_root(node) != 0) {
            pr_err("Reserved memory: unsupported node format, ignoring\n");
            /* break scan */
            return 1;
        }
        found = 1;
        /* scan next node */
        return 0;
    } else if (!found) {
        /* scan next node */
        return 0;
    } else if (found && depth < 2) {
        /* scanning of /reserved-memory has been finished */
        return 1;
    }

    // reserved-memory의 child-node들이 진입 가능
    if (!of_fdt_device_is_available(initial_boot_params, node))
        return 0;
        
    err = __reserved_mem_reserve_reg(node, uname);
    if (err == -ENOENT && of_get_flat_dt_prop(node, "size", NULL))
        fdt_reserved_mem_save_node(node, uname, 0, 0);

    /* scan next node */
    return 0;
}
```
{% endtab %}

{% tab title="\_\_\_reserved\_mem\_reserve\_reg" %}
```c
/**
 * res_mem_reserve_reg() - reserve all memory described in 'reg' property
 */
static int __init __reserved_mem_reserve_reg(unsigned long node,
                         const char *uname)
{
    int t_len = (dt_root_addr_cells + dt_root_size_cells) * sizeof(__be32);
    phys_addr_t base, size;
    int len;
    const __be32 *prop;
    int nomap, first = 1;

    prop = of_get_flat_dt_prop(node, "reg", &len);
    if (!prop)
        return -ENOENT;

    if (len && len % t_len != 0) {
        pr_err("Reserved memory: invalid reg property in '%s', skipping node.\n",
               uname);
        return -EINVAL;
    }

    // 함수의 반환된 값이 NULL이 "아닐" 때 nomap이 true
    nomap = of_get_flat_dt_prop(node, "no-map", NULL) != NULL;

    while (len >= t_len) {
        base = dt_mem_next_cell(dt_root_addr_cells, &prop);
        size = dt_mem_next_cell(dt_root_size_cells, &prop);
        
        if (size &&
            early_init_dt_reserve_memory_arch(base, size, nomap) == 0)
            pr_debug("Reserved memory: reserved region for node '%s': base %pa, size %ld MiB\n",
                uname, &base, (unsigned long)size / SZ_1M);
        else
            pr_info("Reserved memory: failed to reserve memory for node '%s': base %pa, size %ld MiB\n",
                uname, &base, (unsigned long)size / SZ_1M);

        len -= t_len;
        if (first) {
            fdt_reserved_mem_save_node(node, uname, base, size);
            first = 0;
        }
    }
    return 0;
}
```
{% endtab %}

{% tab title="fdt\_reserved\_mem\_save\_node" %}
```c
/**
 * res_mem_save_node() - save fdt node for second pass initialization
 */
void __init fdt_reserved_mem_save_node(unsigned long node, const char *uname,
                      phys_addr_t base, phys_addr_t size)
{
    struct reserved_mem *rmem = &reserved_mem[reserved_mem_count];

    if (reserved_mem_count == ARRAY_SIZE(reserved_mem)) {
        pr_err("not enough space all defined regions.\n");
        return;
    }

    rmem->fdt_node = node;
    rmem->name = uname;
    rmem->base = base;
    rmem->size = size;

    reserved_mem_count++;
    return;
}
```
{% endtab %}

{% tab title="early\_init\_dt\_reserve\_memory\_arch" %}
```c
int __init __weak early_init_dt_reserve_memory_arch(phys_addr_t base,
                    phys_addr_t size, bool nomap)
{
    if (nomap)
        return memblock_remove(base, size);
    return memblock_reserve(base, size);
}
```
{% endtab %}
{% endtabs %}

* **Line 24**: Reserved memory child node일 경우에 해당 루틴이 실행된다.
* **Line 28**: 해당 노드로부터 reg 프로퍼티 값을 읽는다.
  *  \_\_reserved\_mem\_reserve\_reg 진입
  * **Line 7**: 루트로부터 어드레스의 크기, 사이즈의 크기를 넘겨 받는다.
  * **Line 13~14**: reg 프로퍼티를 읽는다. 만약 프로퍼티가 존재하지 않으면, ENOENT를 리턴한다. reg 프로퍼티가 있다는 말은, **정적 예약 메모리**라는 말이고, 프로퍼티가 없다면 **동적 예약 메모리**이란 말이다. 해당 함수는 정적 예약 메모리만 처리한다.
  * **Line 17~21**: 프로퍼티의 길이를 검증한다.
  * **Line 24**: 선택 프로퍼티 no-map이 있는지 확인한다.
  * **Line 26~37**: base와 size를 읽어오고, nomap 프로퍼티가 존재한다면, 해당 영역을 memblock\_remove, 존재하지 않으면 memblock\_reserve한다. 
  * **Line 38~41**: 첫 번째의 경우\(**오류와 상관없이?**\), fdt\_reserved\_mem\_save\_node를 호출한다.
* **Line 29~30**: 동적 예약 메모리를 사용하는 노드라면, fdt\_reserved\_mem\_save\_node를 호출한다. \(단 base, size는 모두 0으로 삽입된다.\)

{% hint style="info" %}
fdt\_reserved\_mem\_save\_node는 정적/동적 예약 메모리 영역 모두 호출하게 된다. 이 함수는 동적 예약 메모리를 지원하기 위한 [패치](https://github.com/iamroot16/linux/commit/3f0c8206644836e4f10a6b9fc47cda6a9a372f9b#diff-603376ac7b8737d94a63e7a29e1574b7R454)에 등장했다.
{% endhint %}

{% tabs %}
{% tab title="\_\_init fdt\_init\_reserved\_mem" %}
```c
void __init fdt_init_reserved_mem(void)
{
    int i;

    /* check for overlapping reserved regions */
    __rmem_check_for_overlap();

    for (i = 0; i < reserved_mem_count; i++) {
        struct reserved_mem *rmem = &reserved_mem[i];
        unsigned long node = rmem->fdt_node;
        int len;
        const __be32 *prop;
        int err = 0;

        prop = of_get_flat_dt_prop(node, "phandle", &len);
        if (!prop)
            prop = of_get_flat_dt_prop(node, "linux,phandle", &len);
        if (prop)
            rmem->phandle = of_read_number(prop, len/4);

        if (rmem->size == 0)
            err = __reserved_mem_alloc_size(node, rmem->name,
                         &rmem->base, &rmem->size);
        if (err == 0)
            __reserved_mem_init_node(rmem);
    }
}
```
{% endtab %}

{% tab title="\_\_reserved\_mem\_alloc\_size" %}
```c
static int __init __reserved_mem_alloc_size(unsigned long node,
    const char *uname, phys_addr_t *res_base, phys_addr_t *res_size)
{
    int t_len = (dt_root_addr_cells + dt_root_size_cells) * sizeof(__be32);
    phys_addr_t start = 0, end = 0;
    phys_addr_t base = 0, align = 0, size;
    int len;
    const __be32 *prop;
    int nomap;
    int ret;

    prop = of_get_flat_dt_prop(node, "size", &len);
    if (!prop)
        return -EINVAL;

    if (len != dt_root_size_cells * sizeof(__be32)) {
        pr_err("invalid size property in '%s' node.\n", uname);
        return -EINVAL;
    }
    size = dt_mem_next_cell(dt_root_size_cells, &prop);

    nomap = of_get_flat_dt_prop(node, "no-map", NULL) != NULL;

    prop = of_get_flat_dt_prop(node, "alignment", &len);
    if (prop) {
        if (len != dt_root_addr_cells * sizeof(__be32)) {
            pr_err("invalid alignment property in '%s' node.\n",
             uname);
            return -EINVAL;
        }
        align = dt_mem_next_cell(dt_root_addr_cells, &prop);
    }

    /* Need adjust the alignment to satisfy the CMA requirement */
    if (IS_ENABLED(CONFIG_CMA)
        && of_flat_dt_is_compatible(node, "shared-dma-pool")
        && of_get_flat_dt_prop(node, "reusable", NULL)
        && !of_get_flat_dt_prop(node, "no-map", NULL)) {
        unsigned long order =
            max_t(unsigned long, MAX_ORDER - 1, pageblock_order);

        align = max(align, (phys_addr_t)PAGE_SIZE << order);
    }

    prop = of_get_flat_dt_prop(node, "alloc-ranges", &len);
    if (prop) {

        if (len % t_len != 0) {
            pr_err("invalid alloc-ranges property in '%s', skipping node.\n",
                   uname);
            return -EINVAL;
        }

        base = 0;
        
        while (len > 0) {
            start = dt_mem_next_cell(dt_root_addr_cells, &prop);
            end = start + dt_mem_next_cell(dt_root_size_cells,
                               &prop);

            ret = early_init_dt_alloc_reserved_memory_arch(size,
                    align, start, end, nomap, &base);
            if (ret == 0) {
                pr_debug("allocated memory for '%s' node: base %pa, size %ld MiB\n",
                    uname, &base,
                    (unsigned long)size / SZ_1M);
                break;
            }
            len -= t_len;
        }

    } else {
        ret = early_init_dt_alloc_reserved_memory_arch(size, align,
                            0, 0, nomap, &base);
        if (ret == 0)
            pr_debug("allocated memory for '%s' node: base %pa, size %ld MiB\n",
                uname, &base, (unsigned long)size / SZ_1M);
    }
    
    if (base == 0) {
        pr_info("failed to allocate memory for node '%s'\n", uname);
        return -ENOMEM;
    }

    *res_base = base;
    *res_size = size;

    return 0;
}
```
{% endtab %}

{% tab title="\_\_reserved\_mem\_init\_node" %}
```c


static int __init __reserved_mem_init_node(struct reserved_mem *rmem)
{
    extern const struct of_device_id __reservedmem_of_table[];
    const struct of_device_id *i;

    for (i = __reservedmem_of_table; i < &__rmem_of_table_sentinel; i++) {
        reservedmem_of_init_fn initfn = i->data;
        const char *compat = i->compatible;

        if (!of_flat_dt_is_compatible(rmem->fdt_node, compat))
            continue;

        if (initfn(rmem) == 0) {
            pr_info("initialized node %s, compatible id %s\n",
                rmem->name, compat);
            return 0;
        }
    }
    return -ENOENT;
}
```
{% endtab %}
{% endtabs %}

* **Line 6**: 등록된 정적 예약 메모리 영역 중 겹치는 부분이 있다면, 경고를 띄운다.
  * 내부 구현은 간단하다. 정렬을 하고, 순회하면서 겹치는 영역을 찾는다.
* **Line 8~19**: 예약 메모리 영역을 순회하면 노드에 핸들 프로퍼티가 있다면, 값을 저장한다.
* **Line 21~22**: size가 0이라면, 동적 예약 메모리 영역이므로, 적절한 영역을 할당해주어야 한다. 따라서 \_\_reserved\_mem\_alloc\_size를 호출한다.
  * \_\_reserved\_mem\_alloc\_size로 진입.
  * 동적 예약 메모리 할당의 핵심인 \_\_reserved\_mem\_alloc\_size으로 진입. 
  * **Line 12~20**: size 프로퍼티의 값을 읽는다. 해당 프로퍼티는 필수이므로, 프로퍼티를 못찾으면  바로 리턴한다.
  * **Line 22~32**: no-map 프로퍼티가 설정되어 있는지 확인, align 프로퍼티가 존재한다면 값을 읽는다.
  * **Line 35~42**: cma 영역이라면, align을 조정할 필요가 있다, 조정되지 않는다면, cma 셋업에 실패하게 된다. 해당 [패치](https://github.com/iamroot16/linux/commit/1cc8e3458b5110253c8f5aaf1890d5ffea9bb7b7#diff-af8d4ef3fff0ccf4fa12f41e389bd29d)를 참고하라.
  * **Line 46~59**: alloc-range 프로퍼티의 값이 존재하면, 그 값을 읽는다.
  * **Line 61~69**: 읽은 range를 대상으로 memblock\_find\_in\_range을 호출하여 빈 영역을 찾는다. 만약 no-map이라면 해당 영역을 memblock\_remove를 하고, 아니라면 memblock\_reserve한다.
  * **Line 73~77**: alloc-range 프로퍼티가 존재하지 않으므로, 모든 영역에 대해서 빈 영역을 찾고, 해당 영역을 memblock\_remove/reserve 한다.
  * **Line 80~88**: 메모리 할당에 실패시 에러 코드를 리턴, 성공시 인자로 들어온 base, size에 할당받은 주소와 사이즈를 대입한다. 
* **Line 24~25**: 오류가 없다면, \_\_reserved\_mem\_init\_node 함수를 호출한다. 해당 함수는 예약 메모리 드라이버 지원 추가를 위한 [패치](https://github.com/iamroot16/linux/commit/f618c4703a14672d27bc2ca5d132a844363d6f5f)에 도입되었다.
  * Line 8: RESERVEDMEM\_OF\_DECLARE로 추가되는 테이블을 순회하며, 지원하는 노드에 대해 초기화 함수를 실행한다.

_&lt;200229\_Memblock 발췌&gt;_

{% code title="RESERVEDMEM\_OF\_DECLARE 정의" %}
```c
#define RESERVEDMEM_OF_DECLARE(name, compat, init)          \
    _OF_DECLARE(reservedmem, name, compat, init, reservedmem_of_init_fn)
```
{% endcode %}

{% code title="RESERVEDMEM\_OF\_DECLARE 사용 사례 중, compat이 ufdt의 compatible과 일치하는 것" %}
```c
kernel/dma/coherent.c|397| RESERVEDMEM_OF_DECLARE(dma, "shared-dma-pool", rmem_dma_setup)
kernel/dma/contiguous.c|281| RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);
```
{% endcode %}

{% tabs %}
{% tab title="\_\_reserved\_mem\_init\_node" %}
```c
static int __init __reserved_mem_init_node(struct reserved_mem *rmem)
{
    extern const struct of_device_id __reservedmem_of_table[];
    const struct of_device_id *i;

    for (i = __reservedmem_of_table; i < &__rmem_of_table_sentinel; i++) {
        reservedmem_of_init_fn initfn = i->data;
        const char *compat = i->compatible;

        if (!of_flat_dt_is_compatible(rmem->fdt_node, compat))
            continue;

        if (initfn(rmem) == 0) {
            pr_info("initialized node %s, compatible id %s\n",
                rmem->name, compat);
            return 0;
        }
    }
    return -ENOENT;
}
```
{% endtab %}

{% tab title="rmem\_cma\_setup" %}
```c
static int __init rmem_cma_setup(struct reserved_mem *rmem)
{
    phys_addr_t align = PAGE_SIZE << max(MAX_ORDER - 1, pageblock_order);
    phys_addr_t mask = align - 1;
    unsigned long node = rmem->fdt_node;
    struct cma *cma;
    int err;

    if (!of_get_flat_dt_prop(node, "reusable", NULL) ||
        of_get_flat_dt_prop(node, "no-map", NULL))
        return -EINVAL;

    if ((rmem->base & mask) || (rmem->size & mask)) {
        pr_err("Reserved memory: incorrect alignment of CMA region\n");
        return -EINVAL;
    }

    err = cma_init_reserved_mem(rmem->base, rmem->size, 0, rmem->name, &cma);
    if (err) {
        pr_err("Reserved memory: unable to setup CMA region\n");
        return err;
    }
    /* Architecture specific contiguous memory fixup. */
    dma_contiguous_early_fixup(rmem->base, rmem->size);

    if (of_get_flat_dt_prop(node, "linux,cma-default", NULL))
        dma_contiguous_set_default(cma);

    rmem->ops = &rmem_cma_ops;
    rmem->priv = cma;

    pr_info("Reserved memory: created CMA memory pool at %pa, size %ld MiB\n",
        &rmem->base, (unsigned long)rmem->size / SZ_1M);

    return 0;
}
```
{% endtab %}

{% tab title="dma\_contiguous\_set\_default" %}
```c
static inline void dma_contiguous_set_default(struct cma *cma)
 {
     dma_contiguous_default_area = cma;
 }
```
{% endtab %}
{% endtabs %}

* **Line 6~10:** 각 예약 메모리 영역을 순회하면서, of\_flat\_dt\_is\_compatible 함수를 호출해서 compatible이 일치하는지 확인한다. 일치하지 않으면 스킵한다.
* **Line 13~17:** initfn 함수를 실행하는데 이 함수는 아래의 열거된 함수와 대응된다.
  * **rmem\_dma\_setup**
  * **rmem\_cma\_setup**

**rmem\_cma\_setup 함수를 훑어보면**

* **Line 9~16:**  노드가 reusable하고 no-map이 아닌지 확인한다. 또한 base와 size의 align을 검증한다. 앞서 봤던 복잡한 조건문이 해당 셋업 함수를 위한 것이다.

```c
if (IS_ENABLED(CONFIG_CMA)
        && of_flat_dt_is_compatible(node, "shared-dma-pool")
        && of_get_flat_dt_prop(node, "reusable", NULL)
        && !of_get_flat_dt_prop(node, "no-map", NULL)) {
        unsigned long order =
            max_t(unsigned long, MAX_ORDER - 1, pageblock_order);

        align = max(align, (phys_addr_t)PAGE_SIZE << order);
    }
    dma_contiguous_default_area = cma;
```

* **Line 18:** cma 테이블에서 정적인 cma struct를 받아 ****초기화 한다.\(**내용 추가 필요!**\)
* **Line 24:** fixup 함수를 호출\(내용은 비어있다\)
* **Line 26~27**: linux,cma-default 프로퍼티가 존재한다면\(해당 예약 메모리 영역이 cma-default 영역이라면\),  dma\_continguos\_set\_default함수를 호출한다. 



  













