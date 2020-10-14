# Scope Stack Allocation (01 Aug 13)

Всем давно известно, особенно в геймдеве, что множественные аллокации это плохо по множеству причин, основные:

- фрагментация памяти
- время аллокаций
- локальность выделяемой памяти

Первая проблема особенно остро стоит на платформах у которых этой самой памяти мало, например, мобильные устройства или консоли. Вторая – на всех платформах, тем или иным образом, потому что время аллокации не определено, стандартный аллокатор может задуматься, по только ему понятным причинам, перед тем как отдать вам память, что ни в какие ворота, когда у вас каждая миллисекунда на счету. Третья же проблема может встать в алгоритмах требовательных к локальности обрабатываемых данных, где нужно уменьшить промахи кэша.

В этой заметке я хотел бы рассказать как победить эти проблемы для временных аллокаций, особенно тех, которые делаются каждый кадр, а уж они-то должны быть быстрыми на сколько это возможно и не фрагментировать кучу совсем.<!--preview-->

Для начала нам понадобится линейный аллокатор. В кругах геймдева, штука очень популярная.

```cpp
class linear_allocator{
  // закрываем конструктор копирования и оператор присваивания
  linear_allocator(const linear_allocator&);
  linear_allocator& operator=(const linear_allocator&);
private:
  char* _bgn;
  char* _end;
  char* _pos;
public:
  // здесь можно задать нужное нам выравнивание
  static const size_t Alignment = 8;

  // функция для выравнивания размера
  static size_t aligned_size(size_t size) {
    return (size + (Alignment - 1)) & ~(Alignment - 1);
  }

  // конструктор
  // принимает размер оперируемой памяти в аллокаторе
  explicit linear_allocator(size_t size)
  : _bgn(NULL), _end(NULL), _pos(NULL)
  {
    _bgn = static_cast<char*>(bm_alloc_aligned(size, Alignment));
    if ( !_bgn )
      throw std::bad_alloc();
    _end = _bgn + size;
    _pos = _bgn;
  }

  // деструктор
  // удаляем выделеную в аллокаторе память
  ~linear_allocator() {
    bm_dealloc_aligned(_bgn);
  }

  // функция выделения памяти
  // линейно выделяет память с верхушки стека
  // и швыряет bad_alloc, если выделить не удалось
  void* allocate(size_t size) {
    char* ret = _pos;
    _pos += aligned_size(size);
    if ( _pos > _end ) {
      _pos = ret;
      throw std::bad_alloc();
    }
    return ret;
  }

  // возвращаем текущую вершину стека
  void* cur_pos() const {
    return _pos;
  }

  // этой функцией можно отмотать наш стек на нужную позицию
  void rewind(void* pos) {
    assert(pos >= _bgn && pos <= _end);
    _pos = static_cast<char*>(pos);
  }
};
```

Откуда взялись `bm_alloc_aligned` и `bm_dealloc_aligned` можно прочитать в моей заметке посвященной выровненным аллокациям.

Класс простой и деревянный, что собственно и нужно для подобного аллокатора. Для примера он выделяет память внутри себя, можно переписать, что бы он брал указатель на нужную память из вне, например из глобальной памяти, но это не суть этой заметки. Всё хорошо, конечно, но пользоваться этим классом неудобно, нужно вручную выделять память, вручную освобождать, при исключениях тоже не забывать отматывать вершину стека и т.д.

С этими мыслями мы подходим к объекту с названием `scope_allocator`. Его функция будет заключаться в автоматическом отматывании верхушки линейного аллокатора при собственном уничтожении. Так же он будет поддерживать вложенность наших линейных аллокаций и вызов деструкторов для C++ классов, которые созданы с помощью него.

```cpp
class scope_allocator{
private:
  // структурка для хранения в связаном списке функций,
  // которые будут звать деструкторы объектов
  struct Finalizer{
    void (*fn)(void* ptr);
    Finalizer* next;
  };

  // шаблонная фукция для вызова нужных деструкторов объектов
  // как раз та функция которую храним в финализере
  template < typename T >
  static void destructor_call(void* ptr){
    static_cast<T*>(ptr)->~T();
  }

  // выделяет память под объект нужного размера
  // и его финализер
  Finalizer* alloc_with_finalizer(size_t size) {
    const size_t fin_size =
      linear_allocator::aligned_size(sizeof(Finalizer));
    void* mem = _allocator.allocate(fin_size + size);
    return static_cast<Finalizer*>(mem);
  }

  // получение указателя на память объекта по его финализеру
  static void* object_from_finalizer(Finalizer* fin) {
    const size_t fin_size =
      linear_allocator::aligned_size(sizeof(Finalizer));
    return reinterpret_cast<char*>(fin) + fin_size;
  }
private:
  linear_allocator& _allocator;
  void* _rewind_pos;
  Finalizer* _finalizers;
private:
  // закрываем конструктор копирования и оператор присваивания
  scope_allocator(const scope_allocator&);
  scope_allocator& operator=(const scope_allocator&);
public:
  // конструктор принимающий линейный аллокатор
  // сохраняем тут позицию отматывания и сам аллокатор
  explicit scope_allocator(linear_allocator& allocator)
  : _allocator(allocator)
  , _rewind_pos(allocator.cur_pos())
  , _finalizers(NULL)
  {
  }

  // деструктор
  // пройдемся по списку финализеров для вызова деструкторов
  // созданных объектов и отмотаем линейный аллокатор на
  // сохраненную позицию
  ~scope_allocator() {
    Finalizer* f = _finalizers;
    while ( f ) {
      (*f->fn)(object_from_finalizer(f));
      f = f->next;
    }
    _allocator.rewind(_rewind_pos);
  }

  // функция для создания POD-объектов.
  // для них не нужно звать деструкторы, по-этому
  // просто конструируем объект на выделеной памяти
  template < typename T >
  T* new_pod() {
    return ::new(_allocator.allocate(sizeof(T))) T;
  }

  // функции для создания C++ объектов.
  // для них уже нужно звать деструктор,
  // по-этому выделяем память не только под сам объект,
  // но и под его финализер и инициализируем его нужной функцией,
  // которая позовет деструктор, при выходе аллокатора из скопа.
  template < typename T >
  T* new_object() {
    Finalizer* fin = alloc_with_finalizer(sizeof(T));
    T* res = ::new(object_from_finalizer(fin)) T;
    fin->fn = &destructor_call<T>;
    fin->next = _finalizers;
    _finalizers = fin;
    return res;
  }
};
```

Отлично же! Понятное дело, что класс нуждается в доработке, например, нужно дописать функцию `new_object` с параметрами, которые будут передаваться конструктору создаваемого класса, но это всё детали. Пробуем что получилось :)

```cpp
struct TestClass{
  int i;
  TestClass () { ::printf("TestClass::TestClass()\n"); }
  ~TestClass() { ::printf("TestClass::~TestClass(%i)\n", i); }
};

int main() {
  // главный аллокатор, может быть создан один на всё приложение
  // или на нужный поток
  linear_allocator la(1024);

  // первый скоп-аллокатор
  scope_allocator sc(la);
  // создаём из него объект
  TestClass* obj1 = sc.new_object<TestClass>();
  obj1->i = 1;
  {
    // второй вложеный скоп-аллокатор
    scope_allocator internal_sc(la);
    // создаём и из него объект
    TestClass* obj2 = sc.new_object<TestClass>();
    obj2->i = 2;

    // при выходе из этой области будет вызван деструктор TestClass
    // и отмотан линейный аллокатор из "internal_sc"
  }
  return 0;

  // а при выходе от сюда из "sc"
}
```

Вывод приложения предсказуем:

```cpp
TestClass::TestClass()
TestClass::TestClass()
TestClass::~TestClass(2)
TestClass::~TestClass(1)
```

P.S.
С необходимыми доработками очень крутая вещь получится, для временных аллокаций и аллокаций на кадре. Все идеи взяты из слайдов доклада компании DICE, ознакомится можно [тут](http://dice.se/publications/scope-stack-allocation).
