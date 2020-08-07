# C++ thread and synchronization:
https://thispointer.com/c-11-multithreading-part-1-three-different-ways-to-create-threads/

## 1. thread.

thread<br/>
|__get_id: return the thread id.<br/>
|__joinable: check if joinable, even if it is done.<br/>
|__join: wait for thread done<br/>
|__detach: detach the thread from the calling thread, allowing them to execute independently.<br/>
|__swap<br/>
|__native_handle<br/>
|__hardware_concurrency<br/>

thread.join(): let the caller thread to wait for the completion of thread.
- joinable: thread started.
- if thread is started and has finished, it is still joinable (if not call a join)
- join can only be called once.

- detach:
detach release the resources for join. So if main thread is not going to wait for the thread done, you need detach it.
if detached and want to wait for thread completion, we need use synchronization.
You can not join after detach.
detached thread is also called background thread or daemon thread.

not calling join, main program exit will call thread destructor and terminate
calling join/detach multiple times will cause crash since the resource are not there anymore

- thread function and passing parameters to thread function
function return value is ignored.
all parameters by thread passed by value
reference parameter must be wrapped using std::ref or std::cref
thread takes:
function pointer
function object
lambda function

use member function of a class, we need pass:
function address and object address.

```cpp
void func(){...}
struct funcobj{
	void operator()(){...}
};

thread mythread(func);
thread mythread((funcobj());
thread mythread([]{...});
```

- std::this_thread: refer to current thread
it has several functions:<br/>
|__get_id<br/>
|__yield<br/>
|__sleep_for<br/>
|__sleep_until<br/>

## 2, mutex:
mutex is a lockable object that is designed to signal when critical section needs exclusive access.
generally for a shared resource updates, we need a lock to guarantee serial access.<br/>
|__lock<br/>
|__unlock<br/>
|__try_lock<br/>
|__native_handler<br/>
mutex itself can be used as lock.<br/>

recursive_mutex: allow a mutex to be acquired multiple time. Only the same thread can do this.

## 3. lock
unique_lock and lock_guard are wrapper class for mutex to avoid forgetting unlock.
lock_guard will lock at construction and unlock at destruction.
unique_lock provides some options:
- deferred_lock
- time locking
- recursive locking
- transfer ownership
- use with condition_variable

unique_lock(mutex): lock at construction. You can check the code implementation.
unique_lock(mutex,defer_lock) do not lock
unique_lock(mutex,try_to_lock): try to lock
unique_lock and lock_guard both unlock at destruction.
lock_guard: you cannot call lock and unlock
unique_lock: you can call lock and unlock.

unique_lock:
The class unique_lock is a general-purpose mutex ownership wrapper allowing deferred locking, time-constrained attempts at locking, recursive locking, transfer of lock ownership, and use with condition variables.
A unique lock is an object that manages a mutex object with unique ownership in both states: locked and unlocked.
This class guarantees an unlocked status on destruction (even if not called explicitly). Therefore it is especially useful as an object with automatic duration, as it guarantees the mutex object is properly unlocked in case an exception is thrown.<br/>
|__lock<br/>
|__try_lock<br/>
|__try_lock_for<br/>
|__try_lock_until<br/>
|__unlock<br/>

example: 
```cpp
std::mutex mtx;           // mutex for critical section

void print_block (int n, char c) {
  // critical section (exclusive access to std::cout signaled by lifetime of lck):
  std::unique_lock<std::mutex> lck (mtx);
  for (int i=0; i<n; ++i) { std::cout << c; }
  std::cout << '\n';
}
```
- each time only one lock get access. the for loop and cout are protected.
- declare the lock ensure only one thread will output at any time.
- function done, the lock is unlocked automatically.
so when spawn several threads, they will one by one gain the access and the output will not mess up.

## 4. condition_variable
condition variable is a kind of event mechanism for signaling between threads.

A condition variable is an object able to block the calling thread until notified to resume.

It uses a unique_lock (over a mutex) to lock the thread when one of its wait functions is called. The thread remains blocked until woken up by another thread that calls a notification function on the same condition_variable object.

Objects of type condition_variable always use unique_lock<mutex> to wait: for an alternative that works with any kind of lockable type, see condition_variable_any

The condition_variable class is a synchronization primitive that can be used to block a thread, or multiple threads at the same time, until another thread both modifies a shared variable (the condition), and notifies the condition_variable.

The thread that intends to modify the variable has to

acquire a std::mutex (typically via std::lock_guard)
perform the modification while the lock is held
execute notify_one or notify_all on the std::condition_variable (the lock does not need to be held for notification)
Even if the shared variable is atomic, it must be modified under the mutex in order to correctly publish the modification to the waiting thread.

Any thread that intends to wait on std::condition_variable has to

acquire a std::unique_lock<std::mutex>, on the same mutex as used to protect the shared variable
either
check the condition, in case it was already updated and notified
execute wait, wait_for, or wait_until. The wait operations atomically release the mutex and suspend the execution of the thread.
When the condition variable is notified, a timeout expires, or a spurious wakeup occurs, the thread is awakened, and the mutex is atomically reacquired. The thread should then check the condition and resume waiting if the wake up was spurious.
or
use the predicated overload of wait, wait_for, and wait_until, which takes care of the three steps above
std::condition_variable works only with std::unique_lock<std::mutex>; this restriction allows for maximal efficiency on some platforms. std::condition_variable_any provides a condition variable that works with any BasicLockable object, such as std::shared_lock.

Condition variables permit concurrent invocation of the wait, wait_for, wait_until, notify_one and notify_all member functions.

condition_variable is more similar to CEvent in win32.
CEvent.notify
CEvent.WaitForSingleObject.
condition_variable<br/>
|__notify_one<br/>
|__notify_all<br/>
|__wait<br/>
|__wait_for<br/>
|__wait_until<br/>

wait(unique_lock)

wait(unique_lock, predicate)

wait: will atomically unlock the lock and block the current thread.

unblock by notify_one, notify_all
spurious unblock: it will reacquire the lock and block again.
predicate: it will also check the predicate

notify_one/notify_all: wakeup the condition_variable.

notify does not need a lock. only wait needs a lock.

```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void print_id (int id) {
  std::unique_lock<std::mutex> lck(mtx);
  while (!ready) cv.wait(lck);
  // ...
  std::cout << "thread " << id << '\n';
}

void go() {
  std::unique_lock<std::mutex> lck(mtx);
  ready = true;
  cv.notify_all();
}

int main ()
{
  std::thread threads[10];
  // spawn 10 threads:
  for (int i=0; i<10; ++i)
    threads[i] = std::thread(print_id,i);

  std::cout << "10 threads ready to race...\n";
  go();                       // go!

  for (auto& th : threads) th.join();

  return 0;
}
```
unique_lock apparently assigns different locks on the same mutex, and it always get access.
it only uses the lock for the CV to create an event.
- create a unique_lock ensure the code is threadsafe. lock used to protect modifying the critical variable ready
- unique_lock in go() does not has any relation with the unique_lock in print_id. so they won't be exclusive to each other.
- cv does not depend on unique_lock. We shall understand it from its name.
- cv.wait block the thread itself (calling thread).
- cv.notify: signal an event.
- once the cv.wait cleared, the thread will lock the locker again.

unique_lock: user can unlock it anytime.
lock_guard: it locks at construction and unlock at destruction.

example:
```cpp
condition_variable cv;
mutex mtx;
bool ready = 0;

void helper()
{
	unique_lock<mutex> lck(mtx);  //lock
	cv.wait(lck, [] {return ready; }); //unlock and block, give other thread to gain access
	lck.unlock();
	cout << "helper running" << endl;
	int i = 65535,j=i;
	while (i--) {
		//cout << hex << setw(4) << i << " ";
		if (i % 16 == 0) {
			//cout << endl;
			std::this_thread::sleep_for(chrono::milliseconds(1));
		}
	}
	cout << "helper done" << endl;
	ready = 1;
	cv.notify_one();//notify waiting thread
}

void wait() {
	unique_lock<mutex> lck(mtx); //try to get the lock
	cv.wait(lck, [] {return ready; });//unlock and block, give other thread chance
	//lck.unlock();
	cout << "wait running.." << endl;
	int timeout = 5000;
	//while (!ready) 
	{
		if (cv.wait_for(lck, chrono::milliseconds(timeout)) == cv_status::timeout) //wait for cv event.
		{ //wait for cv notify or timeout.
			cout << "Wait thread Time out..." << endl;
			cv.notify_one();
			return;
		}
	}
}

int main()
{
	
	std::thread read_thread(helper); //this will run until done.
	std::thread wait_thread(wait);
	read_thread.detach();
	wait_thread.detach();
	ready = 1;
	//lck.unlock();
	cv.notify_all();

	unique_lock<mutex> lck(mtx);//lock
	cv.wait(lck); //unlock and block
	cout << "Thread normal exit..." << endl;
}

```
- helper thread is the working thread.
- the two threads run independently
- helper thread will lock until it gets the CV event and lock again
- we unlock it to let the wait thread to run. Otherwise it will block.
- the wait thread gains the lock until it times out or get the event from helper thread.
- the main thread will block on wait, until we get an event from helper or wait.
so this program:
- it will timeout if helper cannot complete in the timeout period
- it will exit normally if helper completes work in time.

The helper runs less than 5 seconds:
if time out sets as 5000ms:
```
wait running..
helper running
helper done
Thread normal exit...
```
if timeout set 1000ms:
```
wait running..
helper running
Wait thread Time out...
Thread normal exit...
```
when timeout, the helper thread is still running.

how to terminate a thread then?
- a thread may hang forever, can we kill it?

more example:
we want to output firstsecondthird whatever the call sequence is.
```cpp
    mutex m;
    condition_variable cv;
    int ready;
    Foo() {
       ready=0;
    }

    void first(function<void()> printFirst) {
        
        // printFirst() outputs "first". Do not change or remove this line.
        unique_lock<mutex> lck(m);
        ready=0;
        printFirst();
        ready=1;
        cv.notify_one();
    }

    void second(function<void()> printSecond) {
        
        // printSecond() outputs "second". Do not change or remove this line.
        unique_lock<mutex> lck(m);
        //cv.wait(lck,[]{return ready==1;});
        while(ready!=1) cv.wait(lck);
        printSecond();
        ready=2;
		//before notify, better unlock it.
		//notify: the thread to wakeup has to wait until this thread is done.
		
        cv.notify_one();
    }

    void third(function<void()> printThird) {
        
        // printThird() outputs "third". Do not change or remove this line.
        unique_lock<mutex> lck(m);
        //cv.wait(lck,[]{return ready==2;});
        while(ready!=2) cv.wait(lck);
        printThird();
    }
```

- lambda function needs ready variable to be static for capturing, not very clear about it
- use while(..) wait... it is equivalent wait(lck,predicate)
- use [this]{} this as capture
- we change ready, ready is shared variable and need access restriction
- first if not locker protected, this will cause deadlock. (since other thread may change it at the same time).
- need use notify_all so that each thread can receive notification.

Some points that must be taken note of are:

Mutex are used for mutual exclusion i.e to safe gaurd the critical sections of a code.
Semaphone/condition_variable are used for thread synchronisation(which is what we want to achieve here).
Mutex have ownership assigned with them, that is to say, the thread that locks a mutex must only unlock it. Also, we must not unlock a mutex that has not been locked (This is what most programs have got wrong).
If the mutex is not used as said above, the behavior is undefined, which however in our case produces the required result.

	
## future & promise
future can be used with asynch, packaged_task, promise.
Many times we encounter a situation where we want a thread to return a result.

a std::future object internally stores a value that will be assigned in future and 
it also provides a mechanism to access that value i.e. using get() member function. 
But if somebody tries to access this associated value of future through get() function before it is available, then get() function will block till value is not available.

future<br/>
|__share: get shared future.<br/>
|__get: get value<br/>
|__valid: check for valid shared state.<br/>
|__wait: wait for ready.<br/>
|__wait_for: <br/>
|__wait_until<br/>

```cpp
bool is_prime (int x) {
  for (int i=2; i<x; ++i) if (x%i==0) return false;
  return true;
}

int main ()
{
  // call function asynchronously:
  std::future<bool> fut = std::async (is_prime,444444443); 

  // do something while waiting for function to set future:
  std::cout << "checking, please wait";
  std::chrono::milliseconds span (100);
  while (fut.wait_for(span)==std::future_status::timeout)
    std::cout << '.' << std::flush;

  bool x = fut.get();     // retrieve return value

  std::cout << "\n444444443 " << (x?"is":"is not") << " prime.\n";

  return 0;
}
```

- async: call function asynchronously
- The value returned by fn can be accessed through the future object returned (by calling its member future::get).

std::promise is also a class template and its object promises to set the value in future. Each std::promise object has an associated std::future object that will give the value once set by the std::promise object.
future get the value in future
promise set the value in future.

promise shares data with its associated future object. set/get pair
promise<br/>
|__get_future<br/>
|__set_value<br/>
|__set_exception<br/>
|__set_value_at_thread_exit<br/>
|__set_exception_at_thread_exit<br/>
|__swap<br/>

std::async() does following things,
It automatically creates a thread (Or picks from internal thread pool) and a promise object for us.
Then passes the std::promise object to thread function and returns the associated std::future object.
When our passed argument function exits then its value will be set in this promise object, so eventually return value will be available in std::future object

async can also accept: function pointer, function object, lambda function.

```cpp
void print_int (std::future<int>& fut) {
  int x = fut.get();
  std::cout << "value: " << x << '\n';
}

int main ()
{
  std::promise<int> prom;                      // create promise

  std::future<int> fut = prom.get_future();    // engagement with future

  std::thread th1 (print_int, std::ref(fut));  // send future to new thread

  prom.set_value (10);                         // fulfill promise
                                               // (synchronizes with getting the future)
  th1.join();
  return 0;
}
```


