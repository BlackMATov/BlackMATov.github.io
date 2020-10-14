# Аллокации, теги и хуки. Часть 1 (03 Jun 13)

Раз уж начал говорить про память, покажу простой способ навешать теги на аллокации с возможность хука на них, с помощью которого можно сделать чудо вещи, например, трекинг памяти.<!--preview-->

Для начала нам нужны простые функции выделения памяти, которые мы будем использовать вместо системных. Так же понадобятся енумы для тегов аллокаций и действий хипа.

Где-то в хидере в самом низу вашего проекта:

```cpp
namespace bm {
	// теги для аллокаций
	enum E_HEAP_TAGS {
		HEAP_BM_SYSTEM_TAG,
		HEAP_STL_STRINGS_TAG,
		HEAP_STL_CONTAINERS_TAG,
		HEAP_UNKNOWN_TAG
	};

	// енум для действий хипа
	enum E_HEAP_ACTS {
		HEAP_ACT_ALLOC,
		HEAP_ACT_FREE
	};

	// тип функции для хука
	typedef void (*THeapHook)(
		E_HEAP_ACTS act, E_HEAP_TAGS tag,
		void* ptr, size_t size);

	// функции выделения и очищения памяти
	void* Heap_Alloc ( size_t size, E_HEAP_TAGS tag );
	void  Heap_Free  ( void*   ptr, E_HEAP_TAGS tag );

	// функция установки пользовательского хука
	THeapHook Heap_SetHook ( THeapHook hook );
}
```

Реализуем всё то, что объявили выше. Где-то в исходниках:

```cpp
// макросы для системных функций выделения и очистки памяти
#define BM_Sys_Alloc ::malloc
#define BM_Sys_Free  ::free

namespace bm {
	namespace allocs_detail {
		// глобальный текущий хук, по-умолчанию пустой
		static THeapHook heap_hook = NULL;

		// функция вызова текущего хука, будет зваться из наших Heap_Alloc и Heap_Free
		static void CallHeapHook(
			E_HEAP_ACTS act, // действие
			E_HEAP_TAGS tag, // тег
			void* ptr,       // память которую выделяем\удаляем
			size_t size)     // размер куска памяти
		{
			if ( heap_hook )
				heap_hook(act, tag, ptr, size);
		}
	}

	// реализация нашего выделения памяти
	void* Heap_Alloc(size_t size, E_HEAP_TAGS tag) {
		// выделяем память с помощью системной функции и зовём хук
		void* ptr = BM_Sys_Alloc(size);
		allocs_detail::CallHeapHook(HEAP_ACT_ALLOC, tag, ptr, size);
		return ptr;
	}

	// реализация нашего освобождения памяти
	void Heap_Free(void* ptr, E_HEAP_TAGS tag) {
		// зовем сначало хук, пока память жива, потом удаляем системной функцией
		allocs_detail::CallHeapHook(HEAP_ACT_FREE, tag, ptr, 0);
		BM_Sys_Free(ptr);
	}

	// реализация установки пользовательского хука
	THeapHook Heap_SetHook( THeapHook hook ) {
		// устанавливаем новый и возвращаем старый (вдруг кому понадобится)
		using allocs_detail::heap_hook;
		THeapHook last = heap_hook;
		heap_hook = hook;
		return last;
	}
}
```

С функциями закончили. Теперь вместо обычных `malloc` и `free` нужно звать `Heap_Alloc` и `Heap_Free` с нужными тегами, не забывая про "3rd party"-библиотеки, которые в большинстве своем предлагают определять для них функции выделения памяти, если же нет, то придется ручками менять, а лучше просто гнать такие библиотеки в шею, ибо не дело это :)

Ок, с `malloc` и `free` всё понятно, что же делать с `new` и `delete`? Всё просто - переопределяем их на свои:

```cpp
inline void* operator new(size_t size, bm::E_HEAP_TAGS tag)
{ return bm::Heap_Alloc(size, tag); }

inline void* operator new[](size_t size, bm::E_HEAP_TAGS tag)
{ return bm::Heap_Alloc(size, tag); }

inline void* operator new(size_t size)
{ return bm::Heap_Alloc(size, bm::HEAP_UNKNOWN_TAG); }

inline void* operator new[](size_t size)
{ return bm::Heap_Alloc(size, bm::HEAP_UNKNOWN_TAG); }

inline void operator delete(void* ptr, bm::E_HEAP_TAGS tag)
{ bm::Heap_Free(ptr, tag); }

inline void operator delete[](void* ptr, bm::E_HEAP_TAGS tag)
{ bm::Heap_Free(ptr, tag); }

inline void operator delete(void* ptr)
{ bm::Heap_Free(ptr, bm::HEAP_UNKNOWN_TAG); }

inline void operator delete[](void* ptr)
{ bm::Heap_Free(ptr, bm::HEAP_UNKNOWN_TAG); }
```

Теперь:
`new TestClass()` - будет выделять память через наши функции с `HEAP_UNKNOWN_TAG` тегом.
`new (HEAP_BM_SYSTEM_TAG) TestClass()` - с тегом заданным в параметре `new`.
Так же, по вкусу, можно переопределить `nothrow new`, если сие нужно для вашего проекта.

И с этим разобрались. Понятное дело, что вызывать `new` таким образом неудобно, по-этому можно написать немного помогающего кода:

```cpp
namespace bm {
	template < E_HEAP_TAGS TAG >
	class Allocable {
		public:
			void* operator new     (size_t size) { return ::operator new     (size, TAG); }
			void* operator new[]   (size_t size) { return ::operator new[]   (size, TAG); }
			void  operator delete  (void*   ptr) {        ::operator delete  ( ptr, TAG); }
			void  operator delete[](void*   ptr) {        ::operator delete[]( ptr, TAG); }
	};
}
```

Простой классик, наследуясь от которого можно получить нужный тег по-умолчанию, например:

```cpp
class Texture : public Allocable<E_HEAP_TEXTURES_TAG>
{...};

// память будет выделена с тегом E_HEAP_TEXTURES_TAG
Texture* texture = new Texture();
```

В следующей части расскажу как это всё прикрутить к STL контейнерам, ибо их-то аллокации тоже чекать надо.
