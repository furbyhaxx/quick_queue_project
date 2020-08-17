# Quick Multiprocessing Queue

This is an implementation of Quick Multiprocessing Queue for Python and work similar to `multiprocessing.queue` (more
information about `multiprocessing.queue` in 
https://docs.python.org/3/library/multiprocessing.html?highlight=process#pipes-and-queues).


## Install
Last release version of the project to install in: https://pypi.org/project/quick_queue_project/

```
pip install quick-queue
```

## Introduction
The motivation to create this class is due to `multiprocessing.queue` is too slow putting and getting elements 
to transfer data transfer between python processes. 

But if you put or get one list with elements work similar as put or get one single element; this list is getting as 
fast as usually but this has too many elements for process in the subprocess and this action is very quickly.

In other words, Multiprocess queue is pretty slow putting and getting individual data, then QuickQueue wrap several 
data in one list, this list is one single data that is enqueue in the queue than is more quickly than put one 
individual data.

While consumer produce and put lists of elements in queue, subprocesses consume those lists and iterate every element, 
then subprocesses have elements very quickly.

## Quick use
Import:
```python
from quick_queue import QQueue
```

Pseudocode without process:
```python
qq = QQueue()

# << Add here `qq` to new process(es) and start process(es) >>

qq.put("value")
# Put all the values you need

qq.end()
# When end put values call to end() to mark you will not put more values and close QQueue
```

Complete example (it needs `import multiprocessing`):
```python
def _process(qq):
    print(qq.get())
    print(qq.get())
    print(qq.get())

if __name__ == "__main__":

    qq = QQueue()

    p = multiprocessing.Process(target=_process, args=(qq,))
    p.start()

    qq.put("A")
    qq.put("B")
    qq.put("C")

    qq.end()

    p.join()
```
Note: you need to call `end` method to perform remain operation and close queue. If you only want put remain data in
queue, you can call `put_remain`, then you need to call manually to `close` (or `end`, this performs `close` operation
too).

You can put al values in one iterable or several iterables with `put_iterable` method (`put_iterable` perform remain
operation when iterable is consumed; but this not close queue, you need call to `close()` or to `end()` in this case):
```python
def _process(qq):
    print(qq.get())
    print(qq.get())
    print(qq.get())

if __name__ == "__main__":

    qq = QQueue()

    p = multiprocessing.Process(target=_process, args=(qq,))
    p.start()

    qq.put_iterable(["A", "B", "C"])
    qq.put_iterable(["D", "E", "F"])

    qq.end()

    p.join()
```

If you need to use `put` in other process, then you need to initialize values in QQueue with `init`. Due to
Python message pass between process it is not possible share values in the same shared Queue object (at least I have
not found the way) and, by other side, maybe you want to define a different initial values per "put process" to
sensor work calculation.
```python
def _process(qq):
    # Define initial args to this process, if you do not call to init method, then it use default values
    qq.init("""<Defined args>""")

    qq.put("A")
    qq.put("B")
    qq.put("C")

    qq.end()

if __name__ == "__main__":

    qq = QQueue()

    p = multiprocessing.Process(target=_process, args=(qq,))
    p.start()

    print(qq.get())
    print(qq.get())
    print(qq.get())

    p.join()
```

You can use defined args in the main constructor if you pass values. You can get initial args
with `get_init_args` (return a dict with your args) in process where you instanced QQueue,
then in second process you can expand those args in `init` method with `**`.

```python
def _process(qq, init_args):
    qq.init(**init_args)

    qq.put("A")
    qq.put("B")
    qq.put("C")

    qq.end()

if __name__ == "__main__":

    qq = QQueue("""<Defined args>""")

    p = multiprocessing.Process(target=_process, args=(qq, qq.get_init_args()))
    p.start()

    print(qq.get())
    print(qq.get())
    print(qq.get())

    p.join()
```


## About performance
An important fact is the size of list (named here "bucket list") in relation productor and consumers process to have
the best performance:
 * If queue is full, mean consumers are slower than productor.
 * If queue is empty, mean productor is slower than consumers.

Then, best size of bucket list (`size_bucket_list`) is where queue is not full and not empty; for this, I implemented
one sensor to determinate in realtime the `size_bucket_list`, you can enable this sensor if `size_bucket_list` is `None`
(if you define a number in `size_bucket_list`, then you want a constant value to `size_bucket_list` and sensor
disable). by default sensor is enabled (`size_bucket_list=None`), because depend on Hardware in your computer this
`size_bucket_list` value should change, I recommend you test the best performance for your computer modifying
`size_bucket_list` (with `None` and with number value).

You can delimite sensor scope with `min_size_bucket_list` and `max_size_bucket_list` (if `max_size_bucket_list`
is None then is infinite):
```python
qq = QQueue(min_size_bucket_list=10, max_size_bucket_list=1000)
```

To disable the sensor define a size in `size_bucket_list`:
```python
qq = QQueue(size_bucket_list=120)
```


## Performance test
Hardware where the tests have been done:
 * Processor: Intel i5 3.2GHz
 * Operating System: Windows 10 x64

Use `python3 tests\performance_qqueue_vs_queue.py`

Put in a producer process and get in a consumer process N elements with `QuickQueue` and `multiprocessing.queue`:

10,000,000 elements (time: Queue = QuickQueue x 13.28 faster):
```
QuickQueue: 0:00:24.436001 | Queue: 0:05:24.488149
```

1,000,000 elements (time: Queue = QuickQueue x 17.55 faster):
```
QuickQueue: 0:00:01.877998 | Queue: 0:00:32.951001
```

100,000 elements (time: Queue = QuickQueue x 6.32 faster):
```
QuickQueue: 0:00:00.591002 | Queue: 0:00:03.736011
```

## Documentation

### Functions:
 * `QQueue`: Main method to create a `QuickQueue` object configured. Args:
     * `maxsize`: maxsize of bucket lists in queue. If `maxsize<=0` then queue is infinite (and sensor is disabled, I
     recommend always define one positive number to save RAM memory). By default: `1000`
     * `size_bucket_list`: `None` to enable sensor size bucket list (require `maxsize>0`). If a number is defined
                                 here then use this number to size_bucket_list and disable sensor. If `maxsize<=0`
                                 and `size_bucket_list==None` then size_bucket_list is default to `1000;` other wise,
                                 if maxsize<=0 and size_bucket_list is defined, then use this number. By default: `None`
     * `min_size_bucket_list`: (only if sensor is enabled) min size bucket list.
                                     `Min == 1` and `max == max_size_bucket_list - 1`. By default: `10`
     * `max_size_bucket_list`: (only if sensor is enabled) max size bucket list. If `None` is infinite.
                                     By default: `None`

### Class:
This is a class with heritage `multiprocessing.queues.Queue`. Methods overwritten:
 * `put_bucket`: This put in the queue a list of data.
 * `put`: This put in the queue a data wrapped in a list. Accumulate data until size_bucket_list, then put in queue.
 * `put_remain`: Call to enqueue rest values that remains.
 * `put_iterable`: This put in this QQueue all data from an iterable.
 * `end`: Helper to call to put_remain and close queue in one method.
 * `get_bucket`: This get from queue a list of data.
 * `get`: This get from queue a data unwrapped from the list.
 * `qsize`: This return the number of bucket lists (not the number of elements)


## Is useful for you?
Maybe you would be so kind to consider the amount of hours puts in, the great effort and the resources expended in 
doing this project. Thank you.

[![paypal](https://www.paypalobjects.com/en_US/i/btn/btn_donateCC_LG.gif)](https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=PWRRXZ2HETVG8&source=url)
