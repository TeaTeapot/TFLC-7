# Лабораторная работа 7. Преобразование и анализ кода с использованием Clang и LLVM

**Автор:** Гладышев Р.М.  
**Вариант:** Цикл `while`  
**Инструменты:** Ubuntu 24.04, Clang 18, LLVM/opt 18, Graphviz

---

## Постановка задачи

### Общее задание

Познакомиться с инструментами Clang и LLVM, научиться получать AST и LLVM IR для кода на C/C++, применять базовые оптимизации и строить граф потока управления (CFG).

Шаги:
1. Установить Clang, LLVM, opt, Graphviz
2. Скомпилировать C-файл и получить AST и LLVM IR
3. Применить оптимизации (-O2) с помощью opt / clang
4. Построить CFG для оптимизированной программы
5. Проанализировать результат и ответить на контрольные вопросы
6. Выполнить индивидуальное задание


## Общая часть работы

### 1.1 Установка и подготовка среды

```bash
sudo apt install clang llvm graphviz
```

Установлены: `clang 18`, `opt-18`, `graphviz 2.42`.

### 1.2 Исходный код (общая часть)

```c
#include <stdio.h>

int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    printf("%d\n", b);
    return 0;
}
```

### 1.3 Получение AST

```bash
clang -Xclang -ast-dump -fsyntax-only main.c
```

<img width="1210" height="773" alt="02" src="https://github.com/user-attachments/assets/3c3424bf-3057-4e25-9ebe-ff304d6552e2" />


AST отображает полное синтаксическое дерево: `FunctionDecl` для `square` и `main`, узлы `BinaryOperator` для умножения, `CallExpr` для вызова `square(a)` и `printf`.

### 1.4 Генерация LLVM IR

```bash
clang -O0 -S -emit-llvm main.c -o main_O0.ll
clang -O2 -S -emit-llvm main.c -o main_O2.ll
```

В `-O0`:
- Переменные размещены через `alloca`
- Множество операций `load` / `store`
- `square` вызывается как отдельная функция

В `-O2`:
- `square` встроена (`-inline`) и вычислена (`-constprop`)
- Все `alloca`/`load`/`store` удалены (`-mem2reg`, `-dce`)
- Осталось только `printf(25)`

<img width="952" height="799" alt="03" src="https://github.com/user-attachments/assets/52a0503b-6f7e-4fc9-b0ea-e174269a5447" />

<img width="952" height="799" alt="04" src="https://github.com/user-attachments/assets/2fb034fe-96f5-4b56-a656-97bc0f67ee1b" />

<img width="1210" height="773" alt="05" src="https://github.com/user-attachments/assets/cbad6e7c-c95b-4f2b-beb5-ce8378e8092e" />

<img width="1025" height="817" alt="06" src="https://github.com/user-attachments/assets/a261cbe4-24b4-4ec1-9375-69d68d213e43" />


### 1.5 Оптимизация IR и CFG

```bash
opt-18 --passes="dot-cfg" -disable-output main_O2.ll
dot -Tpng .main.dot -o cfg_main.png
```

<img width="1210" height="773" alt="08" src="https://github.com/user-attachments/assets/3b89f42f-d13a-4dd9-be5f-934ea532f91c" />


После `-O2` функция `main` содержит единственный базовый блок. Функция `square` исчезла полностью.

<img width="1280" height="800" alt="09" src="https://github.com/user-attachments/assets/5e1b3a0e-ba79-4438-959a-2220518052d4" />

<img width="1280" height="800" alt="010" src="https://github.com/user-attachments/assets/1812a25d-3e50-488f-a3f2-65afe0b7f2a5" />


---

## Индивидуальная часть работы — Цикл `while`

**Исходный код (`while.c`):**

```c
int main() {
    int i = 0, sum = 0;
    while (i < 10) {
        sum += i;
        i++;
    }
    return sum;
}
```

**Задания:**
1. Построить IR для `-O0`
2. Применить `-indvars`, `-licm`, `-loop-unroll`
3. Построить CFG
4. Указать, в чём отличие `while` от `do-while` в IR

---

### Задание 1. IR для -O0

```bash
clang -O0 -S -emit-llvm while.c -o while_O0.ll
```

<img width="1210" height="773" alt="3" src="https://github.com/user-attachments/assets/3883cdf8-a106-4e4b-99c1-be4c8bbe05e3" />

<img width="1210" height="773" alt="4" src="https://github.com/user-attachments/assets/4c93ea78-7a2f-43f9-a033-99c96ca89dbe" />


**Ключевые наблюдения при `-O0`:**
- 3 блока: `entry` → `header(4)` → `body(7)` → `exit(13)`
- Все переменные (`i`, `sum`) в памяти — через `alloca`
- Условие проверяется **до** тела: `icmp slt → br i1` в блоке 4
- Блок `entry` сразу переходит к `header` (`br label %4`)
- Это структура `while`: *проверка перед выполнением*

### Задание 2. Применение оптимизаций: -indvars, -licm, -loop-unroll

Поскольку `-O0` добавляет атрибут `optnone`, предварительно снимаем его:

```bash
sed 's/optnone //' while_O0.ll > while_no_optnone.ll
```

#### Шаг 1: mem2reg (SSA-перевод — основа для остальных проходов)

```bash
opt-14 --passes="mem2reg" while_no_optnone.ll -S -o while_mem2reg.ll
```
<img width="1210" height="773" alt="6" src="https://github.com/user-attachments/assets/932026bd-6e22-4470-ae3d-7978f388c43f" />

<img width="1210" height="773" alt="7" src="https://github.com/user-attachments/assets/5f72abe4-2a92-4408-88a5-a4abdc61c787" />

<img width="1210" height="773" alt="8" src="https://github.com/user-attachments/assets/f3f698b7-cff5-4214-a053-3cdd8447ed3e" />


После `mem2reg` переменные переведены в SSA-форму: исчезли `alloca`/`load`/`store`, появились `phi`-узлы.

#### Шаг 2: indvars (анализ индукционных переменных)

```bash
opt-18 --passes="mem2reg,indvars" while_no_optnone.ll -S -o while_indvars.ll
```


`-indvars` распознал `i` как индукционную переменную с диапазоном `[0, 10)` и вычислил значение `sum = 0+1+...+9 = 45` на этапе компиляции.

#### Шаг 3: licm (вынос инвариантных вычислений из цикла)

```bash
opt-14 --passes="function(mem2reg,loop-mssa(licm<allowspeculation>))" \
       while_no_optnone.ll -S -o while_licm.ll
```

В данном примере LICM не изменил IR, поскольку тело цикла не содержит инвариантных выражений — все операции зависят от переменной `i`. Пример применения LICM: если бы в цикле было `y = x * 2` (где `x` не меняется), LICM вынес бы это вычисление до цикла.

<img width="740" height="79" alt="9" src="https://github.com/user-attachments/assets/229d5ad0-779a-403a-85d1-c1ca38fcd3ee" />


#### Шаг 4: loop-unroll (развёртывание цикла)

```bash
opt-14 --passes="mem2reg,loop-unroll" while_no_optnone.ll -S -o while_unroll.ll
```

<img width="1210" height="773" alt="10" src="https://github.com/user-attachments/assets/f4d67a8e-6cb9-4c23-ab7c-6bb904e31923" />

<img width="1210" height="773" alt="11" src="https://github.com/user-attachments/assets/36d8507d-2171-4e44-9388-cda025f00438" />

<img width="1280" height="800" alt="12" src="https://github.com/user-attachments/assets/f04d213e-69df-434f-8e01-2e61990a44b7" />

<img width="1280" height="800" alt="13" src="https://github.com/user-attachments/assets/24370930-f6f3-491a-91f0-6d31294216ff" />


`-loop-unroll` полностью развернул цикл (10 итераций) и сложил константное выражение.

#### Сводная таблица эффекта оптимизаций

| Проход      | Эффект на IR цикла `while`                                               |
|-------------|--------------------------------------------------------------------------|
| `mem2reg`   | Удалены `alloca`/`load`/`store`, переменные → SSA с `phi`-узлами        |
| `indvars`   | Распознана инд. переменная `i`, вычислен результат `sum=45` статически  |
| `licm`      | Нет эффекта (нет инвариантных выражений в цикле)                        |
| `loop-unroll` | Цикл полностью развёрнут в 10 последовательных блоков               |

### Задание 3. Построение CFG

```bash
# CFG для O0 (с alloca)
opt-14 --passes="dot-cfg" -disable-output while_O0.ll
dot -Tpng .main.dot -o while_O0_cfg.png

# CFG после mem2reg (SSA-форма)
opt-14 --passes="dot-cfg" -disable-output while_mem2reg.ll
dot -Tpng .main.dot -o while_mem2reg_cfg.png

# CFG после всех оптимизаций
opt-14 --passes="dot-cfg" -disable-output while_all_passes.ll
dot -Tpng .main.dot -o while_allpasses_cfg.png
```

<img width="511" height="402" alt="14" src="https://github.com/user-attachments/assets/0cc973d9-8b26-4cfb-b254-679ced96c552" />

<img width="576" height="615" alt="16" src="https://github.com/user-attachments/assets/33aefeae-1cc0-48b3-8c73-50a68b168cb3" />

<img width="511" height="732" alt="19" src="https://github.com/user-attachments/assets/e5e1e995-2868-4298-b588-dd55bb856bc1" />


### Задание 4. Отличие `while` от `do-while` в IR

**Код `do-while` (`dowhile.c`):**

```c
int main() {
    int i = 0, sum = 0;
    do {
        sum += i;
        i++;
    } while (i < 10);
    return sum;
}
```

**IR `while` после `mem2reg`:**

```llvm
; while: проверка ПЕРЕД телом
  br label %1           ; entry → header
1:                      ; HEADER — сначала условие
  %.01 = phi i32 [ 0, %0 ], [ %5, %3 ]
  %.0  = phi i32 [ 0, %0 ], [ %4, %3 ]
  %2 = icmp slt i32 %.01, 10  ; ← условие проверяется здесь
  br i1 %2, label %3, label %6
3:                      ; BODY — тело выполняется только если условие истинно
  %4 = add nsw i32 %.0, %.01
  %5 = add nsw i32 %.01, 1
  br label %1           ; ← возврат к header
6:
  ret i32 %.0
```

**IR `do-while` после `mem2reg`:**

```llvm
; do-while: тело ПЕРЕД проверкой
  br label %1           ; entry → body (сразу!)
1:                      ; BODY — первая итерация без проверки
  %.01 = phi i32 [ 0, %0 ], [ %3, %4 ]
  %.0  = phi i32 [ 0, %0 ], [ %2, %4 ]
  %2 = add nsw i32 %.0, %.01  ; ← тело идёт первым
  %3 = add nsw i32 %.01, 1
  br label %4
4:                      ; LATCH — проверка условия в конце
  %5 = icmp slt i32 %3, 10  ; ← условие проверяется здесь
  br i1 %5, label %1, label %6  ; ← TRUE → тело, FALSE → выход
6:
  ret i32 %2
```

**Сравнение в таблице:**

| Характеристика              | `while`                                | `do-while`                            |
|-----------------------------|----------------------------------------|---------------------------------------|
| Число базовых блоков        | entry + header + body + exit = 4       | entry + body + latch + exit = 4       |
| Расположение условия        | В **header** (до тела)                 | В **latch** (после тела)              |
| Гарантия выполнения тела    | **Нет** — тело может не выполниться   | **Да** — тело выполнится хотя бы раз |
| Обратная дуга CFG           | `body → header → body`                | `latch → body → latch`               |
| Блок с `icmp`               | Header (первый условный блок)          | Latch (последний блок цикла)          |
| Начало entry                | `br label %header`                     | `br label %body` (сразу в тело)       |

**Вывод:** Ключевое структурное различие — `while` начинается с блока-заголовка (`header`), где `icmp`-инструкция проверяет условие **до** входа в тело. `do-while` начинается сразу с тела, а `icmp` вынесен в отдельный блок-защёлку (`latch`) **после** тела. Это означает, что в `while` возможен ноль итераций, а в `do-while` тело гарантированно выполняется хотя бы один раз. В IR это видно по тому, куда ведёт первый `br` из entry: в `while` — к `icmp`, в `do-while` — к телу цикла.

---

## Выводы

### По индивидуальному заданию

1. В IR (`-O0`) цикл `while` представлен тремя блоками: `entry` → `header` (проверка) → `body` (тело) → обратно к `header` или к `exit`.
2. `mem2reg` переводит переменные цикла в SSA-форму через `phi`-узлы, убирая все `alloca`.
3. `indvars` распознаёт индукционную переменную `i` и при константных границах вычисляет результат статически (`sum = 45`).
4. `licm` не изменяет IR если в теле нет инвариантных выражений; для циклов без побочных эффектов он уже не нужен.
5. `loop-unroll` полностью разворачивает цикл с константными границами, заменяя его последовательностью блоков.
6. Главное отличие `while` от `do-while`: в `while` условие стоит **до** тела (в header), в `do-while` — **после** тела (в latch).

---

## Ответы на контрольные вопросы

**1. Что такое Clang?**
Clang — это компилятор для языков C, C++ и Objective-C, построенный на инфраструктуре LLVM. Его роль: лексический и синтаксический анализ, построение AST, семантический анализ, генерация LLVM IR. Clang выступает «фронтендом» — преобразует исходный код в независимое от платформы промежуточное представление.

**2. Что представляет собой LLVM?**
LLVM (Low Level Virtual Machine) — набор компиляторных технологий с модульной архитектурой. Он предоставляет: оптимизирующий средний уровень (middle-end) для работы с IR, бэкенды для генерации кода под разные платформы (x86, ARM, RISC-V и др.), инструменты `opt`, `llc`, `lld`. Современные компиляторы (Clang, Rust, Swift) используют LLVM как бэкенд.

**3. Чем отличается AST от LLVM IR?**
AST (абстрактное синтаксическое дерево) — структура, близкая к исходному коду: содержит узлы для объявлений, выражений, операторов. Она платформонезависима и хранит информацию о типах. LLVM IR — линеаризованное трёхадресное представление в SSA-форме: инструкции, виртуальные регистры, базовые блоки. IR гораздо ближе к машинному коду и является объектом оптимизаций.

**4. Для чего необходимо промежуточное представление?**
IR позволяет разделить компилятор на независимые уровни: фронтенд (исходный язык → IR) и бэкенд (IR → машинный код). Это даёт возможность: применять оптимизации независимо от языка и платформы, повторно использовать бэкенды для разных языков, анализировать программы алгоритмически.

**5. Что делает инструкция `alloca`?**
`alloca` выделяет память в стековом фрейме текущей функции. Используется для хранения локальных переменных до применения `mem2reg`. После оптимизации `mem2reg` большинство `alloca` заменяются регистрами SSA.

**6. Зачем нужна оптимизация кода?**
Цели: уменьшить количество инструкций, снизить потребление памяти, повысить скорость выполнения. Оптимизации устраняют мёртвый код, подставляют константы, разворачивают циклы, встраивают функции.

**7. Что такое SSA-форма?**
SSA (Static Single Assignment) — форма IR, в которой каждая переменная присваивается ровно один раз. Это упрощает анализ зависимостей данных. Там, где несколько путей управления сходятся, используются `phi`-функции для выбора значения. SSA ускоряет большинство оптимизаций (constant propagation, dead code elimination и др.).

**8. Что такое граф потока управления (CFG)?**
CFG — ориентированный граф, где узлы — базовые блоки (линейные последовательности инструкций без переходов внутри), а рёбра — возможные переходы управления. CFG позволяет анализировать: достижимость блоков, доминирование, циклы, вероятности переходов.

**9. Как представлены арифметические операции в LLVM IR?**
Каждая операция имеет вид `%result = op type %lhs, %rhs`. Например:
- сложение: `%4 = add nsw i32 %.0, %.01`
- умножение: `%5 = mul nsw i32 %0, %0`
- сравнение: `%6 = icmp slt i32 %5, 10`

Флаги типа `nsw` (no signed wrap) сообщают оптимизатору о гарантиях отсутствия переполнения.

**10. Почему функции — отдельные единицы анализа?**
Функция имеет чёткую границу входа (один entry block) и выходов (инструкции `ret`). Это упрощает построение CFG, анализ доминирования и применение оптимизаций. Межпроцедурный анализ (IPA) требует отдельной инфраструктуры (call graph, inlining).

**11. Что происходит с короткой функцией при одном вызове?**
Оптимизация `-inline` встраивает тело функции в место вызова (inlining). После этого `-constprop` / `-sccp` может вычислить результат статически, если аргументы известны. В итоге функция полностью исчезает из IR.

**12. Преимущества IR и CFG над анализом исходного C-кода?**
- IR уже прошёл синтаксический анализ — нет синтаксических вариаций
- SSA-форма делает зависимости явными
- CFG даёт точную модель потока управления для алгоритмов анализа (доминаторы, живые переменные)
- Оптимизации применяются независимо от исходного языка
- Возможна точная оценка достижимости блоков и вероятностей переходов
