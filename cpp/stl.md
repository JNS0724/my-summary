# STL组件

## 六大组件

容器\算法\迭代器\仿函数\adapter\allocator

## allocator

头文件memory分为allocator.h和std_construct.h两个头文件，construct里的constrcut函数用来调用placement new

### std_construct.h

construct() 调用placement new初始化

destory() 用type_trait判断是否需要调用析构函数，需要的话即调用。

### allocator.h

定义了allocator class继承于allocator_base,allocator_base默认就是new_allocator

因此std::allocator默认就是new_allocator,new_allocator调用的是全局的::operator new

## Type_traits

一种类型萃取的技巧, 具体实现是通过模板匹配,获得参数类型信息,根据模板偏特化来进行特殊处理.

## vector

具体实现:

```c++

struct _Vector_impl_data
{
    pointer _M_start; // 使用空间头位置
    pointer _M_finish; // 使用空间尾位置
    pointer _M_end_of_storage; // 可用空间的结尾
//...
}

// 扩容
_M_check_len(size_type __n, const char* __s) const
{
    //...
    // 获得新长度 这里取当前使用的空间大小和需要的大小作比较, 作为扩容的大小.
    const size_type __len = size() + (std::max)(size(), __n);
    return (__len < size() || __len > max_size()) ? max_size() : __len;
    //...
}
```

普通的push_back就是1->2->4的两倍扩容

resize(n):

这里看三个参数:

size: 实际使用的容量

n: 想要填充到这么多

capacity: 数组所占的内存大小

* 如果 n > size, 则扩容到 size + n, 扩容后size = size + n, capacity = size + n;
* 如果 n < size, 则扩容到 size + size, 扩容后 size = n, capacity = size + size;

如果n比原来的capacity小的话,就是size = n,capacity不变,也就是不会缩容.

reverse(n):就是把capacity扩到n,其他不变

不过不管咋说,扩容都是要重新分配内存.

## list

环形双向链表,尾部多一个节点.

插入删除不会使原来的迭代器失效.

## deque

一段段定量连续空间串联起来的队列.

### 初始化

默认创建8块内存块,每个内存块默认512字节,如果一个元素大于512字节,则一个元素占一块.
头尾节点不用.

```c++
_M_initialize_map(size_t __num_elements)
    {
      const size_t __num_nodes = (__num_elements/ __deque_buf_size(sizeof(_Tp))
      + 1); // 需要多少内存块

      this->_M_impl._M_map_size = std::max((size_t) _S_initial_map_size,
        size_t(__num_nodes + 2)); // 默认8块 加2是因为头尾不用
      this->_M_impl._M_map = _M_allocate_map(this->_M_impl._M_map_size); // 申请内存

      // For "small" maps (needing less than _M_map_size nodes), allocation
      // starts in the middle elements and grows outwards.  So nstart may be
      // the beginning of _M_map, but for small maps it may be as far in as
      // _M_map+3.

      _Map_pointer __nstart = (this->_M_impl._M_map
          + (this->_M_impl._M_map_size - __num_nodes) / 2); // 就是头节点的下一个节点
      _Map_pointer __nfinish = __nstart + __num_nodes; // 尾节点的前一个节点

      __try
 { _M_create_nodes(__nstart, __nfinish); }
      __catch(...)
 {
   _M_deallocate_map(this->_M_impl._M_map, this->_M_impl._M_map_size);
   this->_M_impl._M_map = _Map_pointer();
   this->_M_impl._M_map_size = 0;
   __throw_exception_again;
 }

      this->_M_impl._M_start._M_set_node(__nstart);
      this->_M_impl._M_finish._M_set_node(__nfinish - 1);
      this->_M_impl._M_start._M_cur = _M_impl._M_start._M_first;
      this->_M_impl._M_finish._M_cur = (this->_M_impl._M_finish._M_first
     + __num_elements
     % __deque_buf_size(sizeof(_Tp)));
    }
```

### 迭代器

```c++
_Elt_pointer _M_cur; 块内当前位置
_Elt_pointer _M_first; 块内第一个元素位置
_Elt_pointer _M_last; 块内最后一个元素位置
_Map_pointer _M_node; 当前内存块
```

通过这几个指针,迭代器实现在内存块中去跳转.

## stack

只能从一端进出,默认以deque为底层实现.可以更换为list.

## queue

一端进一端出,默认以deque为底层实现,可以更换为list.

## priority_queue

优先队列,通常用来实现最大最小堆.底层实现默认是vector.

## unordered_map

底层实现是hashtable,解决哈希冲突是拉链法,在槽中是头插法.

扩容:初始为槽容量11除以最大负载因子(默认为1.0), 由于结果可能不整除,所以默认都要加上1,然后再取比12大的质数13. 即初始槽数为13

每次扩容都是乘以2,再找比其大的质数.

如果不够放,就用(原有元素个数+已有元素个数 ) / 最大负载因子 ,再去求比其大的质数.

```c++
_M_need_rehash(std::size_t __n_bkt, std::size_t __n_elt,
   std::size_t __n_ins) const
  {
    if (__n_elt + __n_ins > _M_next_resize)
      {
        // If _M_next_resize is 0 it means that we have nothing allocated so
        // far and that we start inserting elements. In this case we start
        // with an initial bucket size of 11.
        double __min_bkts
        = std::max<std::size_t>(__n_elt + __n_ins, _M_next_resize ? 0 : 11)
        / (double)_M_max_load_factor;
        if (__min_bkts >= __n_bkt)
            return { true,
                _M_next_bkt(std::max<std::size_t>(__builtin_floor(__min_bkts) + 1,
                    __n_bkt * _S_growth_factor)) };

        _M_next_resize
        = __builtin_floor(__n_bkt * (double)_M_max_load_factor);
        return { false, 0 };
      }
    else
        return { false, 0 };
  }
```

扩容:

```c++
// Rehash when there is no equivalent elements.
  template<typename _Key, typename _Value,
    typename _Alloc, typename _ExtractKey, typename _Equal,
    typename _H1, typename _H2, typename _Hash, typename _RehashPolicy,
    typename _Traits>
    void
    _Hashtable<_Key, _Value, _Alloc, _ExtractKey, _Equal,
        _H1, _H2, _Hash, _RehashPolicy, _Traits>::
    _M_rehash_aux(size_type __n, std::true_type)
    {
      __bucket_type* __new_buckets = _M_allocate_buckets(__n); // 新的bucket数组
      __node_type* __p = _M_begin(); // 首个节点
      _M_before_begin._M_nxt = nullptr;
      std::size_t __bbegin_bkt = 0;
      while (__p)
    { // 开始迁移该节点
        __node_type* __next = __p->_M_next(); // 先获取下一个节点
        std::size_t __bkt = __hash_code_base::_M_bucket_index(__p, __n); // 计算插入的桶下标
        if (!__new_buckets[__bkt])
            { // 空桶的话,这个_p将会是这个桶链表里的最后一个节点,因此让它指向_M_before_begin.next
              // 也就是迭代器的首个节点,然后自己成为迭代器的_M_before_begin.next,这样保证rehash之
              //  后,所有的非空桶都能串起来
                __p->_M_nxt = _M_before_begin._M_nxt;
                _M_before_begin._M_nxt = __p;
                __new_buckets[__bkt] = &_M_before_begin;
                if (__p->_M_nxt)
                    __new_buckets[__bbegin_bkt] = __p;
                __bbegin_bkt = __bkt;
            }
        else
            { // 非空桶,头插法
                __p->_M_nxt = __new_buckets[__bkt]->_M_nxt;
                __new_buckets[__bkt]->_M_nxt = __p;
            }
        __p = __next;
    }

      _M_deallocate_buckets();
      _M_bucket_count = __n;
      _M_buckets = __new_buckets;
    }
```

## 仿函数 函数对象

通过重载调用操作符模仿函数调用的类,可以称为仿函数 函数对象.
比起普通函数 函数指针,更有抽象性

既有函数调用的功能,又有类的抽象,可以存储自身状态,易扩展.

## 配接器（adapters）

用来修饰容器，仿函数或者迭代器的接口,使原来不相互匹配的两个类可以相互匹配
