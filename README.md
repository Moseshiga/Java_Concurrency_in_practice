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
      **++count;**
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





