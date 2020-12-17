# std::shared_ptr 代码试读（一）：代码结构

## 代码结构

std::shared_ptr的相关代码主要包含shared_ptr，\_\_shared_ptr，\_\_shared_ptr_access，\_\_shared_count还有\_Sp_counted_base这几个class。它们之间的关系见下图。

<img src="C:\Users\chengh\Desktop\shared_ptr.PNG" style="zoom:30%;" />

## shared_ptr

作为主要面向用户的class，shared_ptr将大部分功能都委托给了其父类__shared_ptr，也只保留了一个模板参数\_Tp（这一点和unique_ptr不同，没有将deleter当作模板参数），常用的构造函数主要有：

1. 最常用的：

   ```c++
   template<typename _Yp, typename = _Constructible<_Yp*>>
               explicit
               shared_ptr(_Yp* __p)
   ```

2. 除了传入一个指针作为参数，还可以传入一个deleter或者/以及allocator，这一类构造函数在使用对象池的时候很有用：

   ```c++
   template<typename _Yp, typename _Deleter,
               typename = _Constructible<_Yp*, _Deleter>>
                   shared_ptr(_Yp* __p, _Deleter __d)
   ```

   

   ```c++
    *  __shared_ptr will release __p by calling __d(__p)
    */
    template<typename _Yp, typename _Deleter, typename _Alloc,
        typename = _Constructible<_Yp*, _Deleter, _Alloc>>
           shared_ptr(_Yp* __p, _Deleter __d, _Alloc __a)
   ```

其余拷贝、赋值、移动构造函数不做介绍。

## \_\_shared_ptr_access

该class是\_\_shared_ptr的父类，主要提供了对shared_ptr所存储内容的访问接口operator*() 和operator->()：

```c++
// Define operator* and operator-> for shared_ptr<T>.
template<typename _Tp, _Lock_policy _Lp,
    bool = is_array<_Tp>::value, bool = is_void<_Tp>::value>
        class __shared_ptr_access
        {
            public:
                using element_type = _Tp;
                element_type&
                    operator*() const noexcept;
					...
                element_type*
                    operator->() const noexcept;
				  ...
        };
```
注意这里基于_Tp是不是array和void类型做了偏特化，用到了is_array和is_void这两个类型萃取：

```c++
// Define operator-> for shared_ptr<cv void>.
template<typename _Tp, _Lock_policy _Lp>
    class __shared_ptr_access<_Tp, _Lp, false, true>
    {
	   ...
    };
// Define operator[] for shared_ptr<T[]> and shared_ptr<T[N]>.
template<typename _Tp, _Lock_policy _Lp>
    class __shared_ptr_access<_Tp, _Lp, true, false>
    {
 	  ...
    };
```

两个type trait的实现如下：

```c++
  /// is_array
  template<typename>
    struct is_array
    : public false_type { };
  template<typename _Tp, std::size_t _Size>
    struct is_array<_Tp[_Size]>
    : public true_type { };
  template<typename _Tp>
    struct is_array<_Tp[]>
    : public true_type { };

  // Holds if the template-argument is a void type.
  template<typename _Tp>
    struct __is_void
    {
      enum { __value = 0 };
      typedef __false_type __type;
    };
  template<>
    struct __is_void<void>
    {
      enum { __value = 1 };
      typedef __true_type __type;
    };
```

## \_\_shared_ptr

该class主要包含两个成员，一个element_type*的指针，指向为其所管理的memory，一个\_\_shared_count的对象。其中__shared_count承担了shared_ptr的大部分职责，比如引用计数管理，内存分配（比如常规使用情况下的控制单元的内存分配，以及使用make_shared时的被管理对象的内存分配）。

```c++
template<typename _Tp, _Lock_policy _Lp>
    class __shared_ptr
    : public __shared_ptr_access<_Tp, _Lp>
    {           
        private:
                element_type*	   _M_ptr;         // Contained pointer.
                __shared_count<_Lp>  _M_refcount;    // Reference counter.
};
```
## __shared_count，\_Sp_counted_base，\_Sp_counted_ptr，\_Sp_counted_deleter，以及\_Sp_counted_ptr_inplace

在这个__shared_count中，用到了type erasure技术。鉴于这一技术的特殊性，这里简单对其介绍一下。

### Type erasure

关于Type erasure的介绍，参见《C++ templates》第二版第22章的内容（https://github.com/Walton1128/CPP-Templates-2nd--）：

> 该技术主要被用来桥接基于virtual table的dynamic polymorphism和基于template的static polymorphism。这样既可以免除使用dynamic polymorphism时对公共基类的要求，也可以免除使用static polymorphism时需要在编译期知道具体类型的的要求。当然，该技术也有其不足的地方：由于该技术用到了virtual table，其性能可能会比较接近于dynamic polymorphism（比static polymorphism差一些）。

典型的实现方式如下：

```c++
class Concept
{
    public:
   		virtual ~Concept() = default;
   		virtual void Print() = 0;
   		virtual Concept *Clone() = 0;
  	protected:
    	Concept() = default;
 };

template <typename T> //在子类中多引入了一个或多个template parameter
  class Model : public concept
  {
  	public:
      	//在子类的某一个构造函数中可以再引入一个template parameter
   		template <typename U>     
   		Model(U &&u) : mInstance(std::forward<U>(u)) {}  //此时应使用std::forward
        //也可以在子类的某一个构造函数中直接使用主模板参数T
   		Model(T &&t) : mInstance(std::move(t)) {}    //此时应使用std::move
   		void Print() override { mInstance.Print(); }
  	private:
    	T mInstance;
  };

class Object
{
	public:
  		template <typename T>
  		Object (T&& t) : mConcept(new Model<T>(std::forward<T>(t))) {}  // forwarding constructor
  		~Object() { delete mConcept; }                  // destructor
  		void Invoke() { mConcept->Print(); }
	private:
  		Concept *mConcept;   // 此处通过指针使用dynamic polymorphism
};
```

在shared_ptr中，type erasure的实现与之类似（没有使用forward构造函数），如下：

```c++
template<_Lock_policy _Lp>
    class __shared_count
    {
    	...
    	_Sp_counted_base<_Lp>*  _M_pi;   //指向父类的指针
    };
    
 template<_Lock_policy _Lp = __default_lock_policy>
        class _Sp_counted_base: public _Mutex_base<_Lp>
        {
             ...
                // Called when _M_weak_count drops to zero.
                virtual void
                    _M_destroy() noexcept
                    { delete this; }

                virtual void*
                    _M_get_deleter(const std::type_info&) noexcept = 0;
			...
             _Atomic_word  _M_use_count;     // #shared
             _Atomic_word  _M_weak_count;    // #weak + (#shared != 0)
        };

    // Counted ptr with no deleter or allocator support
    template<typename _Ptr, _Lock_policy _Lp>
        class _Sp_counted_ptr final : public _Sp_counted_base<_Lp>
    {
          ...
          explicit
          _Sp_counted_ptr(_Ptr __p) noexcept   //之所以没有使用forward构造函数，是因为如果已经明确知道参数将是一个指针类型，pass by value是最安全也应该是最高效果的
          : _M_ptr(__p) { }

            virtual void
                _M_destroy() noexcept
                { delete this; }

            virtual void*
                _M_get_deleter(const std::type_info&) noexcept
                { return nullptr; }
		...
         _Ptr             _M_ptr;
    };

    // Support for custom deleter and/or allocator
    template<typename _Ptr, typename _Deleter, typename _Alloc, _Lock_policy _Lp>
        class _Sp_counted_deleter final : public _Sp_counted_base<_Lp>
    {
            
        class _Impl : _Sp_ebo_helper<0, _Deleter>, _Sp_ebo_helper<1, _Alloc>
        {
			...
        };

                    // __d(__p) must not throw.
        _Sp_counted_deleter(_Ptr __p, _Deleter __d) noexcept
            : _M_impl(__p, std::move(__d), _Alloc()) { }

        // __d(__p) must not throw.
        _Sp_counted_deleter(_Ptr __p, _Deleter __d, const _Alloc& __a) noexcept
            : _M_impl(__p, std::move(__d), __a) { }

        virtual void
            _M_destroy() noexcept
            {
				...
            }

        virtual void*
            _M_get_deleter(const std::type_info& __ti) noexcept
            {
				...
            }
        _Impl _M_impl;
    };

    template<typename _Tp, typename _Alloc, _Lock_policy _Lp>
        class _Sp_counted_ptr_inplace final : public _Sp_counted_base<_Lp>
    {
        class _Impl : _Sp_ebo_helper<0, _Alloc>
        {
			...
        };
            
         template<typename... _Args>
            _Sp_counted_ptr_inplace(_Alloc __a, _Args&&... __args)
            : _M_impl(__a)
            {
				...
            }

         // Override because the allocator needs to know the dynamic type
        virtual void
            _M_destroy() noexcept
            {
				...
            }

        private:
        friend class __shared_count<_Lp>; // To be able to call _M_ptr().

        // No longer used, but code compiled against old libstdc++ headers
        // might still call it from __shared_ptr ctor to get the pointer out.
        virtual void*
            _M_get_deleter(const std::type_info& __ti) noexcept override
            {
				...
            }
        _Impl _M_impl;
    };
```
上述代码中，通过使用type erasure技术，用\_Sp_counted_base的三个子类分别处理了std::shared_ptr的三种构造情况：

1. \_Sp_counted_ptr： 用于直接用裸指针对std::shared_ptr(_Yp* __p)进行初始化的情况，此时使用默认的deletor和allocator。
2. \_Sp_counted_deleter： 用于同时指定裸指针和deletor/allocator的情况：std::shared_ptr(_Yp* __p, [](_\_Yp* p){delete p;})。
3. \_Sp_counted_ptr_inplace:  用于通过std::make_shared() 或者std::allocator_shared()构造std::shared_ptr的情况。

更多关于以上三个问题的分析，见std::shared_ptr 代码试读（二）：std::make_shared（尚未完成）。

# 讨论

std::shared_ptr没有像std::unique_ptr那样将deletor的类型用作模板参数。这样做的好处是可以用一个同质容器（homogeneous container）存储所有元素类型相同（deletor可以不同）的std::shared_ptr，从而在使用上给了用户很大自由度。但是，为了实现这一目的，开发者使用了type erasure技术，而type erasure又用到了dynamic polymorphism（virtual函数），这应该是stl中少有几个用到了virtual函数的地方，因为stl的开发者们一直都在尽量避免使用这一可能会带来性能损失的技术。












