# Частичный typedef (12 Jun 13)

В прошлой заметке я показал, как сделать аллокатор для stl контейнеров, но использовать это в сыром виде нереально, ибо такой код сломает руки и глаза. Попытаемся исправить ситуацию.<!--preview--> Сырой вид:

```cpp
typedef std::vector<int, StlAllocator<int, HEAP_STL_CONTAINERS_TAG> > TVec;
TVec vec;
vec.push_back(30);
```

Первое что приходит на ум - написать `typedef` для типа вектора с нашим аллокатором, но компилятор это не переварит :)

```cpp
// логично, но неверно
template < typename T >
typedef std::vector<T, StlAllocator<T, HEAP_STL_CONTAINERS_TAG> > vector;
```

Читаем документацию, стандарт, форумы, и понимаем, что такого в плюсАх просто нет. Окей, придется что-то придумывать. Из головы приходит сразу два решения:

1) наследоваться от вектора и определить аллокатор, аля:

```cpp
template < typename T >
class vector : public std::vector<T, StlAllocator<T, HEAP_STL_CONTAINERS_TAG> > {
	/// ...
};
```

Отметаем такой вариант, так как вектор вообще не предназначался для наследования от него, о чём говорит его не виртуальный деструктор, например.


2) сделать всё это через внешнюю структуру, а там уже сделать typedef, костыльно, конечно, но что ж поделать.

```cpp
// наш "чудо-костыль"
namespace bm {
	template < typename T, typename A = StlAllocator<T, HEAP_STL_CONTAINERS_TAG> >
	struct vector {
		// сам тип вектора
		typedef std::vector<T,A> type;

		// а это для удобства получения типов итераторов,
		// когда не делаешь своего typedef'а на конкретный тип вектора
		typedef typename type::iterator       iterator;
		typedef typename type::const_iterator const_iterator;
	};
}

// пример использования

void foo() {
	bm::vector<int>::type my_vec;
	my_vec.push_back(1);
}
```

Конечно, лишний `::type` режет глаз, но на это придется пойти и ждать поддержки С++11 везде, об этом ниже.

P.S.
В новом стандарте осознали проблему с typedef'ами и ввели решение в стандарт. Выглядит оно так:

```cpp
template < typename T >
using vector = std::vector<T, StlAllocator<T, HEAP_STL_CONTAINERS_TAG>>;

void foo() {
	vector<int> my_vec;
	my_vec.push_back(1);
}
```

Отлично, но... поддержки-то пока везде нет, на данный момент, например, нет поддержки "Type aliases" (как комитет назвал его), в последней версии Visual Studio. Поэтому те, кому нужна кросскомпиляторность, вынуждены использовать костыли описанные выше и ждать :(
