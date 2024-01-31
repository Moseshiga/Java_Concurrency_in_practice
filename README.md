# То, что узнал в книге "Java Concurrency in practice" 

## Описание

Это мой заметки того, что я узнал изкниги "Java Concurrency in practice"

Если Вы автор и считаете, что данный конспект нарушает авторские права - прошу сообщить, я сделаю этот репозиторий
приватным.

# 1. Введение

# 2. Потокобезопастность

Написание потокобезопасного кода — это, по сути, управление доступом к состоянию и, в частности, 
к совместному (shared) мутируемому состоянию (mutable state).

При проектировании потокобезопасных классов хорошие объектно‑ориентированные технические решения: 
инкапсуляция, немутируемость и четкая спецификация инвариантов — будут вашими помощниками.

## 2.1. Что такое потокобезопастность 

Класс является потокобезопасным, если он ведет себя правильно во время доступа из многочисленных 
потоков, независимо от того, как выполнение этих потоков планируется или перемежается рабочей 
средой, и без дополнительной синхронизации или другой координации со стороны вызывающего кода.

Потокобезопасные классы инкапсулируют любую необходимую синхронизацию сами и не нуждаются в 
помощи клиента.

### 2.1.1. Пример: сервлет без поддержки внутреннего состояния
```java
@ThreadSafe
public class StatelessFactorizer implements Servlet {
    public void service(ServletRequest req, ServletResponse resp) {
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      encodeIntoResponse(resp, factors);
    }
}
```

Объекты без поддержки внутреннего состояния всегда являются потокобезопасными

## 2.2. Атомарноть

Сервлет, подсчитывающий запросы без необходимой
синхронизации. Так делать не следует

```java
@NotThreadSafe
public class UnsafeCountingFactorizer implements Servlet {
  private long count = 0;
  public long getCount() { return count; }
  public void service(ServletRequest req, ServletResponse resp) {
      BigInteger i = extractFromRequest(req);
      BigInteger[] factors = factor(i);
      ++count;
      encodeIntoResponse(resp, factors);
    }
}
```

Хотя операция приращения **++count** имеет компактный синтаксис, она не является атомарной 
(atomic), то есть неделимой, а представляет собой последовательность из трех операций: «прочитать,
изменить, записать»

### 2.2.1. Состояние гонки

Наиболее распространенным типом состояния гонки является ситуация «проверить и затем 
действовать», где потенциально устаревшее наблюдение используется для принятия решения о том, что
делать дальше

### 2.2.2. Пример: состояние гонки в ленивой инициализации

```java
@NotThreadSafe
public class LazyInitRace {
  private ExpensiveObject instance = null;
  public ExpensiveObject getInstance() {
    if (instance == null)
      instance = new ExpensiveObject();
    return instance;
  }
}
```

Во время инициализации потока A, поток B может тоже начать инициализировать объект

### 2.2.3. Составные действия

Операции A и B являются атомарными, если, с точки зрения потока, выполняющего операцию A, 
операция B либо была целиком выполнена другим потоком, либо не выполнена даже частично.

Там, где это удобно, используйте существующие потокобезопасные объекты, такие как AtomicLong, 
для управления состоянием вашего класса. Возможные состояния существующих потокобезопасных 
объектов и их переходы в другие состояния легче поддерживать и проверять на потокобезопасность, 
нежели произвольные переменные состояния.

## 2.3. Блокировка

Сервлет, пытающийся кэшировать свой последний результат без адекватной атомарности. Так делать 
не следует

```java
@NotThreadSafe
public class UnsafeCachingFactorizer implements Servlet {
    private final AtomicReference<BigInteger> lastNumber
            = new AtomicReference<BigInteger>();
    private final AtomicReference<BigInteger[]> lastFactors
            = new AtomicReference<BigInteger[]>();
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        if (i.equals(lastNumber.get()))
            encodeIntoResponse(resp, lastFactors.get());
        else {
            BigInteger[] factors = factor(i);
            lastNumber.set(i);
            lastFactors.set(factors);
            encodeIntoResponse(resp, factors);
        }
    }
}
```

Когда в инварианте участвуют многочисленные переменные, поля не являются независимыми: значение 
одного ограничивает допустимые значения другого. Следовательно, при обновлении одного необходимо 
обновлять другие в той же атомарной операции.

Для сохранения непротиворечивости состояний обновляйте родственные переменные состояния в единой 
атомарной операции.

### 2.3.1. Внутренние замки

синхронизированный блок, состоящий из ссылки на объект-замок (lock), и блока кода, который будет 
им защищен.

```java
synchronized (lock) {
 // Обратиться к защищаемому замком совместному состоянию либо его
 // изменить
}
```

Внутренние замки в Java действуют как взаимоисключающие замки — мьютексы (mutual exclusion 
locks). Когда поток A пытается приобрести замок, которым владеет поток B, он должен ждать или 
блокировать продвижение до тех пор, пока B его не освободит. Если B не освободит замок никогда, 
то A будет ждать вечно.

Сервлет, кэширующий последний результат с неприемлемо
слабой конкурентностью. Так делать не следует

```java
ThreadSafe
public class SynchronizedFactorizer implements Servlet {
    @GuardedBy("this") private BigInteger lastNumber;
    @GuardedBy("this") private BigInteger[] lastFactors;
    public synchronized void service(ServletRequest req,
                                     ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        if (i.equals(lastNumber))
            encodeIntoResponse(resp, lastFactors);
        else {
            BigInteger[] factors = factor(i);
            lastNumber = i;
            lastFactors = factors;
            encodeIntoResponse(resp, factors);
        }
    }
}
```

### 2.3.2. Повторная входимость

Код, запертый взаимной блокировкой, так как внутренние замки не являются повторно входимыми

```java
public class Widget {
    public synchronized void doSomething() {
        ...
    }
}

public class LoggingWidget extends Widget {
    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
```

## 2.4. Защита состояния с помощью замков

Все обращения к мутируемой переменной состояния должны выполняться с удержанием одного и того же 
замка. Только тогда переменная защищена этим замком.

Каждая совместная мутируемая переменная должна быть защищена только одним замком. Дайте 
сопроводителям четкое понимание, какой это замок.

Для каждого инварианта, который включает более одной переменной, все переменные, участвующие в 
инварианте, должны быть защищены тем же замком.

## 2.5. Живучесть и производительность 


При реализации политики синхронизации не поддавайтесь искушению пожертвовать простотой (что 
может поставить безопасность под угрозу) ради производительности.

Избегайте удержания блокировки во время длительных вычислений или операций, таких как сетевой 
или консольный ввод‑вывод.

Эти два способа записи означают одно и то же:

```java
public void swap() {
   synchronized (this)
   {
       //...логика метода
   }
}

public synchronized void swap() {
    //...логика метода
}
```

Обновленный код, в котором кешируется последнее число и его делители с счетчиками для статистики

```java
@ThreadSafe
public class CachedFactorizer implements Servlet {
    @GuardedBy("this") private BigInteger lastNumber;
    @GuardedBy("this") private BigInteger[] lastFactors;
    @GuardedBy("this") private long hits;
    @GuardedBy("this") private long cacheHits;
    public synchronized long getHits() { return hits; }
    public synchronized double getCacheHitRatio() {
        return (double) cacheHits / (double) hits;
    }
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = null;
        synchronized (this) {
            ++hits;
            if (i.equals(lastNumber)) {
                ++cacheHits;
                factors = lastFactors.clone();
            }
        }
        if (factors == null) {
            factors = factor(i);
            synchronized (this) {
                lastNumber = i;
                lastFactors = factors.clone();
            }
        }
        encodeIntoResponse(resp, factors);
    }
}
```

# 3. Совместное использование объектов

Синхронизация также имеет еще один важный аспект: видимость памяти (memory visibility). Мы 
обеспечим потокам во время изменения объекта возможность видеть внесенные другим потоком 
изменения и опубликуем объекты с помощью синхронизации (явной или встроенной в библиотечные 
классы)

## 3.1. Видимость

Совместное использование переменных без синхронизации. Так делать не следует

```java
public class NoVisibility {
    private static boolean ready;
    private static int number;
    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready)
                Thread.yield();
            System.out.println(number);
        }
    }
    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```

Без синхронизации компилятор, процессор и рабочая среда могут запутать порядок выполнения 
операций. Не стоит ожидать естественного порядка действий памяти в недостаточно 
синхронизированных многопоточных программах.

Нужно применять правильную синхронизацию, когда данные используются потоками совместно.

### 3.1.1 Устаревшие данные

Класс NoVisibility (3.1.) продемонстрировал один из путей появления устаревших данных (stale 
data), которые видит читающий поток, если синхронизация не используется всякий раз при обращении 
к переменной.

Устаревшие значения могут вызывать сбои в безопасности или жизнеспособности: неожиданные 
исключения, поврежденные структуры данных,неточные вычисления и бесконечные циклы.

Непотокобезопасный мутируемый держатель целого числа

```java
@NotThreadSafe
public class MutableInteger {
    private int value;
    public int get() { return value; }
    public void set(int value) { this.value = value; }
}
```

Потокобезопасный мутируемый держатель целого числа

```java
@ThreadSafe
public class SynchronizedInteger {
    @GuardedBy("this") private int value;
    public synchronized int get() { return value; }
    public synchronized void set(int value) { this.value = value; }
}
```

### 3.1.2. Неатомарные 64-разрядные операции

Если несколько одновременно записывают и считывают 64-разрядные переменные (long, double), то 
возможна ситуация когда можно получить верхние 32 бита одного значения и нижние 32 бита другого

### 3.1.3. Блокировка и видимость

Чтобы обеспечить видимость актуальных значений совместных волатильных переменных, 
синхронизируйте читающие и пишущие потоки на общем замке.

### 3.1.4. Волатильные переменные

Переменная volatile для компилятора и рабочей среды является совместной, то есть операции над 
ней не будут переупорядочены с другими операциями в памяти. Волатильные переменные не кэшируются 
в регистрах или кэшах, где данные скрыты от других процессоров, поэтому их чтение всегда 
возвращает самый последний результат операций записи.

На большинстве современных процессорных архитектур волатильные чтения ненамного дороже 
неволатильных

Используйте волатильные переменные только тогда, когда они упрощают реализацию и проверку 
политики синхронизации. И избегайте их использования, когда диагностика правильности требует 
утонченных рассуждений о видимости. Волатильные переменные должны обеспечивать видимость их 
собственного состояния, состояния объекта, на который они ссылаются, или важного события 
жизненного цикла (например, инициализации или выключения).

 Типичное использование волатильных переменных:

```java
volatile boolean asleep;
...
    while (!asleep)
        countSomeSheep();
```

Совет по отладке: в серверном приложении всегда указывайте переключатель командной строки JVM-
server во время вызова JVM даже для разработки и тестирования. Серверная JVM выполняет больше 
оптимизационных задач, чем клиентская JVM, например извлекает из цикла переменные, которые в нем 
не модифицируются. Код, который может показаться работающим в рабочей среде (клиентская JVM), 
может нарушиться в среде развертывания (серверная JVM). Если бы в листинге 3.4 мы забыли 
объявить переменную asleep как волатильную, то серверная JVM могла бы вынуть проверку из цикла 
(превратив его в бесконечный цикл), а клиентская JVM — не могла бы. Бесконечный цикл, который 
проявляется при разработке, стоит намного дешевле, чем тот, который обнаруживает себя в процессе 
эксплуатации кода.

Волатильные переменные удобны и часто используются в качестве флажка завершения, прерывания или 
статуса. Однако механизм volatile недостаточно силен, для того чтобы сделать операцию приращения 
(count++) атомарной в многопоточной среде. Поэтому:

Блокировка может гарантировать как видимость, так и атомарность, а волатильные переменные 
гарантируют только видимость.

Использование волатильных переменных оправданно при следующих
условиях:
- записи в переменную не зависят от ее текущего значения, либо есть гарантия, что значения
  переменной обновляются только одним потоком;
- переменная не участвует в инвариантах с другими переменными состояния;
- при обращении к переменной заранее не требуется блокировка.

## 3.2. Публикация и ускользание

Позволяем ускользнуть внутреннему мутируемому состоянию.
Так делать не следует

```java
class UnsafeStates {
    private String[] states = new String[] {
        "AK", "AL" ...
    };
    public String[] getStates() { return states; }
}
```

Любой вызывающий массив states элемент кода может изменить его содержимое. В данном случае 
предположительно приватный объект стал публичным.

Когда класс ThisEscape публикует слушателя EventListener, он неявно публикует и окаймляющий его 
экземпляр ThisEscape, потому что экземпляры внутреннего класса содержат скрытую ссылку на него.

Неявное разрешение ссылке this ускользнуть. 
Так делать не следует

```java
public class ThisEscape {
    public ThisEscape(EventSource source) {
        source.registerListener(
        new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
         });
     }
}
```

### 3.2.1. Приемы безопасного конструирования

Не позволяйте ссылке this ускользнуть во время конструирования.

Объект находится в предсказуемом и непротиворечивом состоянии только при полноценности его 
конструктора. В противном случае возникает риск публикации неполно сконструированного объекта, 
даже если публикация является последней инструкцией в конструкторе. Объект с ускользнувшей 
ссылкой this считается ненадлежаще сконструированным

Не позволяйте ссылке this ускользнуть во время конструирования.

Ссылка this не должна ускользать из потока до тех пор, пока конструктор не возвратится. Ссылка 
this может храниться где-то конструктором, при условии что она не используется другим потоком до 
завершения конструирования.

Использование фабричного метода для предотвращения ускользания ссылки this во время 
конструирования

```java
public class SafeListener {
    private final EventListener listener;
    private SafeListener() {
        listener = new EventListener() {
            public void onEvent(Event e) {
                doSomething(e);
            }
        };
    }
    public static SafeListener newInstance(EventSource source) {
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

## 3.3. Ограничение одним потоком

Сам язык не содержит механизмов ограничения объекта одним потоком. Такое ограничение станет 
элементом проекта вашей программы. Языковые и ядерные библиотеки предоставляют механизмы 
поддержки ограничения — локальные переменные и класс ThreadLocal, — но ответственность за их 
реализацию лежит на разработчике.

### 3.3.1. Узкоспециальное ограничение одним потоком

Более прочными формами ограничения одним потоком являются ограничение стеком и ThreadLocal.

### 3.3.2. Ограничение стеком

При ограничении стеком (stack confinement) объект может быть достигнут только через локальные 
переменные.

### 3.3.3. ThreadLocal

Класс ThreadLocal позволяет ассоциировать каждое значение в потоке с объектом, владеющим этим 
значением. Он предоставляет методы доступа get и set, поддерживающие отдельную копию значения 
для каждого потока, который его использует, поэтому get возвращает самое последнее значение, 
переданное в set из выполняющегося в настоящее время потока.

Используя класс ThreadLocal для хранения соединения JDBC, как в ConnectionHolder, вы обеспечите 
каждому потоку свое собственное соединение.


Использование класса ThreadLocal для ограничения одним потоком:

```java
private static ThreadLocal<Connection> connectionHolder = new ThreadLocal<Connection>() {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};
public static Connection getConnection() {
    return connectionHolder.get();
}
```
Значения TreadLocal хранятся в самом объекте Thread, и когда поток терминируется, они могут быть
собраны сборщиком мусора.

## 3.4. Немутируемость

Немутируемые объекты всегда являются потокобезопасными.

### 3.4.1. Финальные поля

Объявляйте поля приватными, если они не нуждаются в большей видимости, и финальными — если они не
должны быть мутируемыми.

### 3.4.2 Пример: использование volatile для публикации немутируемых объектов

Немутируемый держатель для кэширования числа и его множителей

```java 
@Immutable
class OneValueCache {
    private final BigInteger lastNumber;
    private final BigInteger[] lastFactors;
    public OneValueCache(BigInteger i, BigInteger[] factors) {
        lastNumber = i;
        lastFactors = Arrays.copyOf(factors, factors.length);
    }
    public BigInteger[] getFactors(BigInteger i) {
        if (lastNumber == null || !lastNumber.equals(i))
            return null;
        else
            return Arrays.copyOf(lastFactors, lastFactors.length);
    }
}
```

Состояния гонки при доступе или обновлении родственных переменных могут быть устранены в 
немутируемом держателе даже без блокировки: когда поток приобретет на него ссылку, другой поток 
уже не сможет изменить его состояние. Чтобы обновить переменные, потребуется создать новый 
держатель, но потоки, работающие с предыдущим держателем, продолжат видеть его в непротиворечивом 
состоянии.

Кэширование последнего результата с помощью изменчивой ссылки на немутируемый держатель объекта

```java
@ThreadSafe
public class VolatileCachedFactorizer implements Servlet {
    private volatile OneValueCache cache = new OneValueCache(null, null);
    public void service(ServletRequest req, ServletResponse resp) {
        BigInteger i = extractFromRequest(req);
        BigInteger[] factors = cache.getFactors(i);
        if (factors == null) {
            factors = factor(i);
            cache = new OneValueCache(i, factors);
        }
        encodeIntoResponse(resp, factors);
    }
}
```

## 3.5. Безопасная публикация

Публикация объекта без надлежащей синхронизации. Так делать не следует

```java
// Небезопасная публикация
public Holder holder;
public void initialize() {
    holder = new Holder(42);
}
```

Вы будете удивлены, но из‑за проблем видимости держатель Holder может показаться другому потоку в 
противоречивом состоянии, даже если его инварианты были правильно установлены его конструктором! 
Именно ненадлежащая публикация заставит другой поток наблюдать частично сконструированный объект.

### 3.5.1. Ненадлежащая публикация: хорошие объекты становятся плохими

Не стоит надеяться на целостность частично сконструированных объектов. Наблюдающий поток может увидеть сначала объект в противоречивом состоянии, а затем внезапное изменение состояния. Если держатель Holder в 3.5. будет опубликован так, как в листинге 3.14, при вызове метода assertSanity возникнет ошибка AssertionError!

Риск возникновения сбоя при ненадлежащей публикации

```java
public class Holder {
    private int n;
    public Holder(int n) { this.n = n; }
    public void assertSanity() {
        if (n != n)
            throw new AssertionError("Эта инструкция является ложной.");
    }
}
```

Проблема здесь не в самом классе Holder, а в его публикации. Holder может быть сделан 
невосприимчивым к ненадлежащей публикации, если объявить поле n финальным, что сделает класс 
Holder немутируемым

Поскольку синхронизация не использовалась для того, чтобы сделать класс Holder видимым для других 
потоков, другие потоки увидят либо устаревшее значение в поле holder (пустую ссылку null или 
другое), либо актуальное значение ссылки на holder, но устаревшие значения состояния Holder, либо 
устаревшее значение при первом чтении поля, а в следующий раз — более актуальное значение. 

Да, без достаточной синхронизации могут происходить очень странные вещи, если данные используются 
потоками совместно.

### 3.5.2. Немутируемые объекты и безопасность при инициализации

Немутируемые объекты могут безопасно использоваться потоками без дополнительной синхронизации, 
даже когда синхронизация для их публикации не используется.

