# std::shared_ptr 代码试读（二）：std::make_shared

正如在“std::shared_ptr 代码试读（一）：代码结构”中最后所讲的那样，std::shared_ptr的构造有三种情况，而且中最为精妙，最为高效，也最广为推荐的一种构造方式就是std::make_shared。因此关于std::shared_ptr构造的代码，本文以std::make_shared为例进行介绍。

## std::make_shared的构造过程

为了分析std::make_shared的代码，先来看一下使用std::make_shared时的对象构造过程：

假设有如下程序：

```C++
#include <iostream>
#include <memory>
using namespace std;
struct A{
    int a;
    A(int input):a(input){;}
};
int main(){
    std::shared_ptr<A> a = std::make_shared<A>(0);
}
```

那么在对象a的构造过程中会依次调用如下函数：

首先std::make_shared会将其实现委托给std::allocate_shared。

```c++
//call stack #4
template<typename _Tp, typename... _Args>   
    inline shared_ptr<_Tp>
    make_shared(_Args&&... __args)
    {
        typedef typename std::remove_cv<_Tp>::type _Tp_nc;
        return std::allocate_shared<_Tp>(std::allocator<_Tp_nc>(),
                std::forward<_Args>(__args)...);
    }
```

std::allocate_shared与std::make_shared功能基本相同，唯一的不同是可以为std::allocate_shared指定allocator，比如在std::make_shared中就为std::allocate_shared指定了默认的std::allocator.

```c++
// call stack #3
template<typename _Tp, typename _Alloc, typename... _Args>  
    inline shared_ptr<_Tp>
    allocate_shared(const _Alloc& __a, _Args&&... __args)
    {
        return shared_ptr<_Tp>(_Sp_alloc_shared_tag<_Alloc>{__a},
                std::forward<_Args>(__args)...);
    }
```

然后在std::allocate_shared中，调用了shared_ptr的构造函数，此时为了在shared_ptr的诸多构造函数中选择适用于allocate_shared的构造函数，使用了tag dispatch技术（_Sp_alloc_shared_tag）。

```C++
 // call stack #2
template<typename _Alloc, typename... _Args> 
        shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args)
        : __shared_ptr<_Tp>(__tag, std::forward<_Args>(__args)...)
        { }
```

通过tag dispatch技术选出了上面的shared_ptr构造函数，该构造函数继续通过tag dispatch技术调用了特定的__shared_ptr:

```c++
// call stack #1         
template<typename _Alloc, typename... _Args>   
                __shared_ptr(_Sp_alloc_shared_tag<_Alloc> __tag, _Args&&... __args)
                : _M_ptr(), _M_refcount(_M_ptr, __tag, std::forward<_Args>(__args)...)
                { _M_enable_shared_from_this_with(_M_ptr); }
```

在这个函数中首先对指向数据存储区内存的指针\_M_ptr进行了默认初始化，然后会在_M_refcount的构造中对\_M_ptr进行赋值。在选择适用于make_shared的\_M_refcount的构造函数时，再次使用了tag dispatch。

```c++
// call stack #0        
template<typename _Tp, typename _Alloc, typename... _Args>   
            __shared_count(_Tp*& __p, _Sp_alloc_shared_tag<_Alloc> __a,
                    _Args&&... __args)
            {
                typedef _Sp_counted_ptr_inplace<_Tp, _Alloc, _Lp> _Sp_cp_type;
                typename _Sp_cp_type::__allocator_type __a2(__a._M_a);
                auto __guard = std::__allocate_guarded(__a2);
                _Sp_cp_type* __mem = __guard.get();
                auto __pi = ::new (__mem)
                    _Sp_cp_type(__a._M_a, std::forward<_Args>(__args)...);
                __guard = nullptr;
                _M_pi = __pi;
                __p = __pi->_M_ptr();
            }
```
在大概知晓了std::make_shared的构造过程之后，下面来解释为什么本文开头会说它是“最为精妙，最为高效，也最广为推荐的”shared_ptr构造方式。



## 为什么说std::make_shared是“最为精妙，最为高效，也最广为推荐的”？

### std::shared_ptr的内存结构

首先shared_ptr大概包含以下数据单元：指向data field的element_type \*类型的指针，以及一个间接的包含了\_M_use_count，\_M_weak_count的\_\_shared_count（在某些情况下它可能还包含一个deletor对象和一个allocator对象，这一区域被称为control block，\__shared_count中包含一个指向这个control block的指针）。

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/shared_ptr_00.png" alt="image-20201217145658450" style="zoom: 30%;" />



### 常规的std::shared_ptr构造

通常data field和control_block是分离的，比如当我们通过如下代码创建一个shared_ptr时：

```c++
std::shared_ptr<int> a(new int);
```

首先，我们为int类型new出了一块内存，然后在构造\__shared_count的时候，又会为control block new出一块内内存，第二次内存分配代码如下：

```c++
        template<typename _Ptr>
            explicit
            __shared_count(_Ptr __p) : _M_pi(0)
            {
                __try
                {
                    _M_pi = new _Sp_counted_ptr<_Ptr, _Lp>(__p);
                }
                __catch(...)
                {
                    delete __p;
                    __throw_exception_again;
                }
            }
			...
            _Sp_counted_base<_Lp>*  _M_pi;
```
为了构建一个std::shared_ptr对象，却进行了两次内存分配，而且第二次内存分配分配的内存还比较小，这一方面会影响程序性能，另一方面还会大大增加内存碎片产生的可能性。

### 使用std::make_shared的std::shared_ptr构造

std::make_shared的精妙之处就在于，它将std::shared_ptr构造中的两次内存分配降低到了一次。这会对提供程序性能和降低内存碎片都有帮助。

<img src="https://github.com/Walton1128/STL-soruce-code-read/blob/main/Figures/shared_ptr_01.png" alt="image-20201217153423778" style="zoom: 30%;" />

其具体实现过程需要参考// call stack #0 中的代码和后文中_Sp_counted_ptr_inplace的相关代码。

在//call stack #0中的\_\_shared_ptr里，会通过萃取技术为\_Sp_counted_ptr_inplace开辟出一块内存，其中_Sp_counted_ptr_inplace唯一的数据成员是\_Impl类型的\_M_impl。下面来看\_Sp_counted_ptr_inplace中直接以及间接包含的信息：

1. 由于\_Sp_counted_ptr_inplace的父类是\_Sp_counted_base，而\_Sp_counted_base里有\_M_use_count和\_M_weak_count两个成员，因此\_Sp_counted_ptr_inplace间接的包含了\_M_use_count和_M_weak_count。
2. 由于\_M_impl继承自 _Sp_ebo_helper<0, _Alloc>，而 _Sp_ebo_helper<0, _Alloc>是一个为了使用空基类优化（EBCO）而引入的辅助类，因此\_Sp_counted_ptr_inplace还间接的包含了一个allocator对象。
3. 由于\_M_impl还有一个\_\_gnu_cxx::__aligned_buffer<_Tp> _M_storage成员，而\_\_gnu_cxx::__aligned_buffer<_Tp>包含的是一个大小和经过内存对其后的\_Tp的大小相同的char数组，其目的是用来存储\_Tp，因此\_Sp_counted_ptr_inplace还间接包含了一个\_Tp。

上述1和2对应于control block，3对应于data fiels。因此在//call stack #0中，通过类型萃取std::__allocate_guarded为\_Sp_counted_ptr_inplace开辟内存，就等于同时为data field和control block开辟了内存。这也正是std::make_shared的精妙所在。

```c++
template<typename _Tp, typename _Alloc, _Lock_policy _Lp>
    class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp>
{
    class _Impl : _Sp_ebo_helper<0, _Alloc>
    {
        typedef _Sp_ebo_helper<0, _Alloc>	_A_base;

        public:
        explicit _Impl(_Alloc __a) noexcept : _A_base(__a) { }

        _Alloc& _M_alloc() noexcept { return _A_base::_S_get(*this); }

        __gnu_cxx::__aligned_buffer<_Tp> _M_storage;
    };

    public:
    using __allocator_type = __alloc_rebind<_Alloc, _Sp_counted_ptr_inplace>;

    // Alloc parameter is not a reference so doesn't alias anything in __args
    template<typename... _Args>
        _Sp_counted_ptr_inplace(_Alloc __a, _Args&&... __args)
        : _M_impl(__a)
        {
            // _GLIBCXX_RESOLVE_LIB_DEFECTS
            // 2070.  allocate_shared should use allocator_traits<A>::construct
            allocator_traits<_Alloc>::construct(__a, _M_ptr(),
                    std::forward<_Args>(__args)...); // might throw
        }

    ~_Sp_counted_ptr_inplace() noexcept { }

    virtual void
        _M_dispose() noexcept
        {
            allocator_traits<_Alloc>::destroy(_M_impl._M_alloc(), _M_ptr());
        }

    // Override because the allocator needs to know the dynamic type
    virtual void
        _M_destroy() noexcept
        {
            __allocator_type __a(_M_impl._M_alloc());
            __allocated_ptr<__allocator_type> __guard_ptr{ __a, this };
            this->~_Sp_counted_ptr_inplace();
        }

    private:
    friend class __shared_count<_Lp>; // To be able to call _M_ptr().

    // No longer used, but code compiled against old libstdc++ headers
    // might still call it from __shared_ptr ctor to get the pointer out.
    virtual void*
        _M_get_deleter(const std::type_info& __ti) noexcept override
        {
            auto __ptr = const_cast<typename remove_cv<_Tp>::type*>(_M_ptr());
            // Check for the fake type_info first, so we don't try to access it
            // as a real type_info object. Otherwise, check if it's the real
            // type_info for this class. With RTTI enabled we can check directly,
            // or call a library function to do it.
            if (&__ti == &_Sp_make_shared_tag::_S_ti()
                    ||
#if __cpp_rtti
                        __ti == typeid(_Sp_make_shared_tag)
#else
                        _Sp_make_shared_tag::_S_eq(__ti)
#endif
                   )
                	return __ptr;
                return nullptr;
           }   
    _Tp* _M_ptr() noexcept { return _M_impl._M_storage._M_ptr(); }
    _Impl _M_impl;
};
```


## 讨论

在std::make_shared的实现中，用到了tag dispatch、EBCO等技术，虽然最终已经将new内存的次数从两次降低到一些，但是由于其成员比std::uniqe_ptr多了atomic类型（硬件支持原子操作的话）的\_M_use_count和\_M_weak_count，因此无论是性能（虽然atomic已经实现无锁了，但是还是比较慢）还是内存占用，都要比std::unique_ptr差很多。所有在实践中，在能用std::unqiue_ptr的地方尽量使用std::unique_ptr，如果非要使用std::shared_ptr，在能够对其进行std::move的时候，就不要对其进行拷贝。

