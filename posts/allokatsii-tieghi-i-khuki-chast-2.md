# Аллокации, теги и хуки. Часть 2 (10 Jun 13)

Как и обещал в прошлой заметке - напишу как сделать аллокатор для stl контейнеров с поддержкой тегов.<!--preview-->

Заходим в любую документацию по C++, лучше всего в стандарт, конечно, но можно куда-нибудь, где по нагляднее, например [сюда](http://www.cplusplus.com/reference/memory/allocator), и реализуем всё что нужно для стандартного аллокатора.  Пишем.

```cpp
namespace bm {
	// в параметры шаблона добавляем тип тега
	template < typename T, E_HEAP_TAGS TAG >
	class StlAllocator {
		// адаптация для const типов
		template < typename U >
		struct AdaptT { typedef U type; };
		template < typename U >
		struct AdaptT<const U> { typedef U type; };
	public:
		// типы требующиеся для контейнеров и пользовательских контейнеров
		typedef typename AdaptT<t>::type value_type;
		typedef value_type*              pointer;
		typedef value_type&              reference;
		typedef const value_type*        const_pointer;
		typedef const value_type&        const_reference;
		typedef size_t                   size_type;
		typedef ptrdiff_t                difference_type;

		// функции получения указателя из ссылки
		// опять же требуется стандартом
		pointer       address(reference       x) const { return &x; }
		const_pointer address(const_reference x) const { return &x; }

		// 1*
		template < typename U >
		struct rebind { typedef StlAllocator<U,TAG> other; };

		// Конструкторы по-умолчанию, копирования,
		// копирования из другого типа, но того же тега и деструктор.
		StlAllocator() {}
		StlAllocator(const StlAllocator&) {}
		template < typename U >
		StlAllocator(const StlAllocator<U,TAG>&) {}
		~StlAllocator() {}

		// собственно функция выделения памяти.
		// используем наш перегруженный оператора new с тегом
		pointer allocate(size_type count, const void* = NULL) {
			void* memory = ::operator new(count * sizeof(T), TAG);
			return static_cast<pointer>(memory);
		}

		// функция освобождения памяти.
		// используем наш перегруженный оператор delete с тегом
		void deallocate(pointer ptr, size_type) {
			::operator delete(ptr, TAG);
		}

		// конструирование объекта в памяти.
		// используем обычный placement new
		void construct(pointer ptr, const T& val) {
			::new(ptr) value_type(val);
		}

		// уничтожение объекта из памяти.
		// зовем вручную деструктор
		void destroy(pointer ptr) {
			ptr->~value_type();
		}

		// максимальное количество элементов, которое может выделить аллокатор.
		size_type max_size() const {
			size_type count = size_type(-1) / sizeof(T);
			return count > 0 ? count : 1;
		}
	};

	// внешние операторы сравнения аллокаторов

	template < typename T1, typename T2, E_HEAP_TAGS TAG >
	inline bool operator == (const StlAllocator<T1,TAG>&, const StlAllocator<T2,TAG>&) {
		return true;
	}

	template < typename T, E_HEAP_TAGS TAG, typename OtherAllocator >
	inline bool operator == (const StlAllocator<T,TAG>&, const OtherAllocator&) {
		return false;
	}

	template < typename T1, typename T2, E_HEAP_TAGS TAG >
	inline bool operator != (const StlAllocator<T1,TAG>&, const StlAllocator<T2,TAG>&) {
		return false;
	}

	template < typename T, E_HEAP_TAGS TAG, typename OtherAllocator >
	inline bool operator != (const StlAllocator<T,TAG>&, const OtherAllocator&) {
		return true;
	}
} // namespace bm
```

1*) Шаблон для ребинда текущего типа аллокатора на другой тип. Используется в "node-base" контейнерах, аля `std::map`, `std::list` и т.д. Нужен потому что в этих контейнерах память выделяется не под пользовательский тип, а под ноду, где хранится этот тип. Что-то типа: `struct Node { Node* prev; Node* next; T user_type; };`

Добавлю еще немного о том, почему тэг передаётся в виде параметра шаблона, а не в конструкторе и не хранится в самом объекте аллокатора. Так исторически сложилось, что стандарт только рекомендует учитывать, что аллокатор может иметь стейт. (в С++11 это исправили как могли, но до многих новые плюсЫ еще не дошли), поэтому использование стейтов в аллокаторе практически невозможно, ибо на разных реализациях они будут по-разному себя вести. Передавая тег в качестве параметра шаблона, мы обходим эту проблему, но так же лишаемся возможности копировать, присваивать, обменивать и т.д. контейнеры с разными типами тегов.

Проверяем что получилось:

```cpp
void stl_test() {
	typedef std::vector<int, StlAllocator<int, HEAP_STL_CONTAINERS_TAG> > TVec;
	TVec vec(10, 5);
	vec.reserve(30);

	typedef std::list<int, StlAllocator<int, HEAP_STL_CONTAINERS_TAG> > TList;
	TList lst;
	lst.push_back(1);
}
```

Отлично, то что и было нужно. Теперь можно повешать хук на всё это и посмотреть что и куда у вас аллочит :)
