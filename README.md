# <a name="Home"></a> Concurrency

## Содержание:
- [Обзор](#Overview)
- [Thread](#Thread)
- [Runnable](#Runnable)
- [Callable and FutureTask](#Callable)
- [Executor](#Executor)
- [Executor Service](#ExecutorService)
- [ScheduledExecutorService](#ScheduledExecutorService)
- [ForkJoinPool](#ForkJoinPool)

## [↑](#Home) <a name="Overview"></a> Обзор
Дано: Базовый класс HelloWorld
```java
public class App {

    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }

}
```
Запустив программу на выполнение, мы увидим в консоли фразу "Hello, World!".
Теперь, давайте добавим в метод main ещё одну строчку кода:
```java
throw new IllegalStateException("Something wrong");
```
При выполнении в консоли увидим что-то вроде:
```
Exception in thread "main" java.lang.IllegalStateException: Something wrong
	at ru.test.App.main(App.java:9)
```
А вот это уже интереснее. Тут есть два главных слова: **thread** и **main**.
Поиска в гугле "java api thread" находим ссылку на Java API: [java.lang.Thread](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.html).
Тут важно понять то, что когда запускается JVM, а в ней наша программа, у нас всегда есть хотя бы один поток. И как правило поток, в котором начинается выполнение нашей программы, называется **main**.
Интересно, что у класса **java.lang.Thread** есть несколько статических методов, которые могут помочь понять, что вообще происходит "под капотом".
Например:
```java
for (Thread thread : Thread.getAllStackTraces().keySet()) {
	System.out.println(thread);
}
```
Оказывается, при запуске Java программы запускается даже не один, а несколько потоков!
Но почему тогда программа завершается? Если мы внимательно читали, мы помним фразу из Java API для класса Thread:
```
All threads that are not daemon threads have died
```
Отыскав нужный метод, чуть модифицируем наш код:
```java
for (Thread thread : Thread.getAllStackTraces().keySet()) {
	if (!thread.isDaemon()) {
		System.out.println(thread);
	}
}
```
И действительно, из всех потоков только main наш является не демон потоком. Поэтому, когда он завершается, то завершается и выполнение всех остальных потоков. А с этим завершается и выполнение программы.

У Oracle есть целый раздел, посвящённый многопоточности: "[Lesson: Concurrency](https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html)".

## [↑](#Home) <a name="Thread"></a> Thread
Итак, в языке java поток описывается классом Thread. Он не является самим потоком, но позволяет управлять потоком, который он представляет.
Поток, как мы видели, может быть демон потоком, а может и не быть.
**Daemon Threads** (Демон потоки) - это служебные потоки. Они не должны выполнять занятие каких-либо ресурсов, т.к. их остановка может произойти в любой момент времени и корректное завершение работы не гарантируется. Поэтому, стоит избегать использования каких либо ресурсов во избежании нарушения целостности данных.
**java.lang.ThreadGroup** - Потоки организованы в группы. Каждая поток присоединён к какой-либо группе. Это позволяет получить список потоков группы. Группы могут образовывать иерархию по типу Родительский элемент - дочерние элементы. 
Пример получения названия группы текущего потока:
```java
System.out.println(Thread.currentThread().getThreadGroup());
```

**Thread.State** - у потока есть состояние.
Узнать его очень просто, достаточно выполнить:
```java
Thread.currentThread().getState()
```
Thread.State - это enum со следующими значениями:
- NEW : Поток только что создан)
- RUNNABLE : Поток запустили (но не говорит, что он сейчас выполняется)
- BLOCKED : Блокирован другим потоком (ожидает монитор, занятый другим потоком)
- WAITING : Ожидает другой поток (методы wait и notify)
- TIMED_WAITING : Временное ожидание (методы wait и sleep с указанием времени)

Создать и запустить поток легко:
```java
Thread thread = new Thread();
thread.start();
```
Если мы посмотрим, то thread.start вызывает метод **thread.run**, который обращается к внутреннему полю target, где лежит некое **Runnable**.
Сам запуск потока так просто не посмотреть, т.к. это низкоуровневый механизм, поэтому метод start указан как native.

## [↑](#Home) <a name="Runnable"></a> Runnable
Итак, Thread описывает некий выполняющийся поток. Поток можем начать, для этого есть метод **start**. Это должно запустить что-то. Поэтому, при создании потока мы можем указать то, что должно запускаться. Для описания этого существует интерфейс **[java.lang.Runnable](https://docs.oracle.com/javase/7/docs/api/java/lang/Runnable.html)**.
Да, мы должны реализовать некоторую логику, иимплементируя данный интерфейс. 
Например:
```java
public static void main(String[] args) {
	Runnable task = new Runnable() {
		public void run() {
			System.out.println("Hello, World!");
		}
	};
	Thread thread = new Thread(task);
	thread.start();
}
```
Важно, что если мы посмотрим внимательнее на код Runnable, то увидим аннотацию над интерфейсом: **@FunctionalInterface**. То есть для описания Runnable можем использовать лямбда выражения:
```java
public static void main(String[] args) {
	Runnable task = () -> { System.out.println("Hello, World!"); };
	Thread thread = new Thread(task);
	thread.start();
}
```
Пустые скобки показывают, что у нас нет входных аргументов.



## [↑](#Home) <a name="Callable"></a> Callable and FutureTask
У Runnable есть брат блезнец - **[java.util.concurrent.Callable](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Callable.html)**.
Отличается от Runnable тем, что он возвращает результат.
Кроме того, просто так Callable нельзя использовать, т.е. Thread в конструктор принимает только Runnable. Для того, чтобы Callable использовать в Thread необходимо использовать его как часть класса **[java.util.concurrent.FutureTask](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/FutureTask.html)**.
**FutureTask** - представляет из себя задачу (Task), которая будет выполнена где-то в будущем (Future).
FutureTask может быть создан как по Callable, так и по Runnable.
FutureTask реализует интересный интерфейс - **RunnableFuture**. Он совмещает в себе интерфейс **Runnable** (что позволяет использовать его в Thread) и интерфейс **Future**.

Пример:
```java
public static void main(String[] args) {
	Callable<String> task = () -> {
		String printedText = "Hello, World!";
		System.out.println(printedText);
		return printedText;
	};
	FutureTask<String> futureTask = new FutureTask(task);
	Thread thread = new Thread(futureTask);
	thread.start();
	try {
		System.out.println("Printed: " + futureTask.get());
	} catch (InterruptedException | ExecutionException e) {
		e.printStackTrace();
	}
}
```
Как мы видим, мы обязаны обработать два разных исключения: **InterruptedException** и  **ExecutionException**.
Первое связано с тем, что когда мы вызываем get, то мы останавливаем выполнение текущего потока до того времени, пока не получим данные из futureTask. То есть выполнение становится синхронным. Соответственно, мы можем в этот момент получить сообщение о том, что наш поток должен прервать своё выполнение, поэтому мч должны обработать Interrupt.
Второе исключение связано с тем, что Callable предоставляется метод call, который обязывает обрабатывать Exception. Т.е. при выполнении может возникнуть исключения ситуация, о которой мы должны позаботиться.

У FutureTask есть так же есть полезные методы, которые позволяют:
- isDone() - узнать, выполнена ли задача
- isCancelled() - узнать, не отменена ли задача
- cancel() - отменить выполнение задачи

Так же есть перегруженный метод get, которому можно сообщить, сколько мы собираемся ждать ответ от потока (что обяжет нас обработать ещё и TimeoutException):
```java
try {
	System.out.println("Printed: " + futureTask.get(1L, TimeUnit.SECONDS));
} catch (InterruptedException | ExecutionException e) {
	e.printStackTrace();
} catch (TimeoutException e) {
	e.printStackTrace();
}
```

Объект futureTask в методе cancel принимает параметр типа boolean. Если он будет true, то перед отменой задачи будет вызван Thread.interrupt. Это нужно для того, чтобы задача смогла как-то обработать тот факт, что сейчас её собираются отменить. Например, откатить транзакцию, закрыть ресурсы, выполнить логирование и т.п.

Если мы хотим использовать Runnable для FutureTask, то мы должны выполнить:
```java
FutureTask<String> futureTask = new FutureTask(task, null);
```

Тогда метод get вернёт null.
Но мы можем указать здесь любое другое значение, которое вернётся в случае успешного завершениея futureTask. Например:
```java
FutureTask<Boolean> futureTask = new FutureTask(task, true);
```
В случае успеха, futureTask.get вернёт true.

Важно помнить, что при выполнении get по отменённой задаче мы получим: **java.util.concurrent.CancellationException**.
Отменить уже выполненную задачу невозможно.

## [↑](#Home) <a name="Executor"></a> Executor
Начиная с Java 1.5 люди поняли, что как-то свалено в одну кучу создание потоков, логика их выполнения, отправление вообще задач в потоки. Хотелось как-то это всё разделить.
И выразили это желание в виде интерфейса [Executor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html).
Соответственно, интерфейс **Executor** имеет всего один метод **execute**, который на вход принимает runnable.
Предполагается, что любая реализация может по какой-то причине отклонить выполнение задачи и бросить RuntimeException: **RejectedExecutionException**.

У данного интерфейса есть реализация - **ThreadPoolExecutor**.
Пример инициализации ThreadPoolExecutor'а:
```java
int startPoolSize = 1;
int maximumPoolSize = 2;
long keelAliveTime = 1;
BlockingQueue<Runnable> workQueue = new ArrayBlockingQueue<>(1);
Executor executor;
executor = new ThreadPoolExecutor(startPoolSize, maximumPoolSize, keelAliveTime, TimeUnit.SECONDS, workQueue);
```
То есть у ThreadPoolExecutor - это представление некоторого пула потоков, у которого есть начальный размер и максимальный. Если потоков не будет хватать, то будут создаваться новые. Если поток простаивает, то после истечения времени жизни он будет убит. Ну и каждый пул должен где-то хранить задачи. Соответственно, мы должны указать блокирующуяся очередь, на основе которой данный пул будет работать.

Для того, чтобы отправить Runnable на выполнение в поток достаточно:
```java
Runnable task = () -> {
	try {
		Thread.currentThread().sleep(3000);
	} catch (InterruptedException e) {
		System.out.println("Interrupted");
	}
	System.out.println("Hello World");
};
for (int i = 0; i < 10; i++) {
	System.out.println(i);
	executor.execute(task);
}
```
Будет выброшено RejectedExecutionException:
> Exception thrown by an {@link Executor} when a task cannot be accepted for execution.

Это отличный пример того, что может произойти плохого. Мы получим исключение:
```
java.util.concurrent.RejectedExecutionException: Task ru.test.App$$Lambda$1/990368553@3d075dc0 rejected from java.util.concurrent.ThreadPoolExecutor@214c265e[Running, pool size = 2, active threads = 2, queued tasks = 1, completed tasks = 0]
```
Как видно, проблема проявляется на 4той итерации. При лимите в 2 потока первые две итерации отправили потоки на выполнение. Размер очереди у нас один. То есть очередь заполнилась полностью на третьей итерации. А четвёртая итерация падает с ошибкой, т.к. пул потоков пуст, очередь полностью забита и задачу некуда деть.
Тут важно то, что статус = Running.

То есть работа регулируется двумя очередям:
- Очередь потоков (регулируется размером пула)
- Блокирующаяся очередь (размер очереди)

ThreadPoolExecutor позволяет так же указать:
- ThreadFactory, требует реализовать:
```Thread newThread(Runnable r)```
- RejectedExecutionHandler, требует реализовать:
```void rejectedExecution(Runnable r, ThreadPoolExecutor executor)```

## [↑](#Home) <a name="ExecutorService"></a> Executor Service
На самом деле, все реализации Executor'а так же являются реализацией другого интерфейса, а именно: [java.util.concurrent.ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html).
Он позволяет управлять временем жизни задач и отслеживать прогресс задач.
Для отправки задач используются следующие варианты:
- <T> Future<T>	submit(Callable<T> task)
- <T> Future<T>	submit(Runnable task, T result)
- Future submit(Runnable task)

Таким образом, в отличии от Executor, ExecutorService возвращает Future, от которого можно дожидаться результата.

Так же предоставляются методы прекращения работы:
- shutdown
- shutdownNow (возврщает список Runnable, которые лежали в очереди на исполнение в момент выполнения метода shutdownNow)

Важно не путать ошибку при сабмите задач при остановленном Executor'е. Вместо **Running** мы увидим **Shutting down**.

Есть так же более мягкий способ остановки:
- awaitTermination(long timeout, java.util.concurrent.TimeUnit unit)

Данный метод приостанавливает текущий поток и ждёт, пока выполнятся все потоки в Executor'е. Если наступает таймаут, а потоки ещё работают - Executor будет остановлен.

Так же интересные варианты:
- invokeAll (возвращает список Future)
- invokeAny (возвращает первый выполненный Future)

Оба метода имеют вариант обычный и с указанием таймаута. Если таймаут наступает - всё что не выполнено становится cancelled.

Позволяет использовать следующие удобные статичные методы для создания ExecutorService'ов:
```java
// Only one thread with runnable queue with size: Integer.MAX_VALUE
ExecutorService executor = Executors.newSingleThreadExecutor(); // keepAlive = 0
// SynchronousQueue doesn't has capacity. Each put wait unless another thread is trying to remove it
ExecutorService executor2 = Executors.newCachedThreadPool(); //keepAlive = 60 sec
// Normal queus (LinkedBlockingQueue) with size
ExecutorService executor3 = Executors.newFixedThreadPool(4); // keepAliveTime 0
```

## [↑](#Home) <a name="ScheduledExecutorService"></a> ScheduledExecutorService
ExecutorService может выполнять задачи и по расписанию.
Для этого служит [ScheduledExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ScheduledExecutorService.html).
Его методы возвращают не просто Future, а **ScheduledFuture**.
Метод schedule позволяет отправить задачу, которая будет отдана на выполнение через указанный промежуток времени. Соответственно, можем передать или Callable или Runnable.
Например:
```java
schedule(Callable<V> callable, long delay, TimeUnit unit)
```
А так же доступны ещё два метода:
- ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)
- ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)

Первый метод выполняет комманду через промежуток времени, а второй выполняет комманду через промежуток времени после завершения выполнения прошлой задачи.
Пример использования:
```java
public static void main(String[] args) throws InterruptedException {
	// Only one thread with runnable queue with size: Integer.MAX_VALUE
	ScheduledExecutorService executor = Executors.newScheduledThreadPool(5);

	Runnable task = () -> {
        try {
            Thread.currentThread().sleep(3000);
        } catch (InterruptedException e) {
            System.out.println("Interrupted");
            return;
        }
		System.out.println("Hello World");
	};

	ScheduledFuture future = executor.schedule(task, 5L, TimeUnit.SECONDS);
	executor.shutdown();
	while (future.getDelay(TimeUnit.SECONDS) != 0) {
		System.out.println(future.getDelay(TimeUnit.SECONDS));
		Thread.currentThread().sleep(1000);
	}
}
```

## [↑](#Home) <a name="ForkJoinPool"></a> ForkJoinPool
Не менее интересной реализацией ExecutorService является [ForkJoinPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html).
Например:
```java
ForkJoinPool pool = new ForkJoinPool(4);
Integer result = pool.invoke(new Fibonacci(40));
```
И сама задача:
```java
class Fibonacci extends RecursiveTask<Integer> {
	final int n;

	Fibonacci(int n) {
		this.n = n;
	}

	@Override
	protected Integer compute() {
		if (n <= 1) {
        	return n;
        }
		Fibonacci f1 = new Fibonacci(n - 1);
		f1.fork();
		Fibonacci f2 = new Fibonacci(n - 2);
		return f2.compute() + f1.join();
	}
}
```
