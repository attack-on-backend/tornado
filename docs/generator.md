# Attack on Tornado - 生成器 🌪










<extoc></extoc>

## 前言

在上一章中 , 我们提到了可以通过协程来避开回调地狱的问题 , 我们这一章先放下对协程的疑惑 , 先来聊一聊生成器

生成器是在 `Python 2.2, PEP 255` 中首次引入的 , 生成器实现了迭代器协议 , 所以我们可以说生成器是迭代器的构造器 , 通过生成器我们可以在循环中计算下一个值时不会浪费内存 , 也就是可以为我们提供惰性计算

我们来自己实现一个 `range` 为例 :

非惰性计算 , 一次性生成 , 你需要有足够大的内存存储结果序列

```python
def eager_range(up_to):
    """Create a list of integers, from 0 to up_to, exclusive."""
    sequence = []
    index = 0
    while index < up_to:
        sequence.append(index)
        index += 1
    return sequence
```

惰性计算 , 生成器方式

```python
def lazy_range(up_to):
    """Generator to return the sequence of integers from 0 to up_to, exclusive."""
    index = 0
    while index < up_to:
        yield index
        index += 1
```

惰性计算 , 闭包方式

```python
def cell_range(up_to):
    """Closure to return the sequence of integers from 0 to up_to, exclusive."""
    index = 0
    def inner():
        nonlocal index
        while index < up_to:
            index += 1
            return index
    return inner
```

对于闭包而言实际上是一次调用完毕的概念 , 而对于生成器而言是暂停代码执行的概念


## PyGenObject

在 `Python` 中 , 生成器的实现就是 `PyGenObject` , 我们以 `Python 3.6.3` 为例 , 来看看它的源代码

`Include/genobject.h` , 13~33行

```C
/* _PyGenObject_HEAD defines the initial segment of generator
   and coroutine objects. */
#define _PyGenObject_HEAD(prefix)                                           \
    PyObject_HEAD                                                           \
    /* Note: gi_frame can be NULL if the generator is "finished" */         \
    /* _frame: PyFrameObject 
        PyFrameObject 是 Python 对 x86 平台上栈帧的模拟,
        同样也是 Python 字节码的执行环境, 也就是当前的上下文
    */
    struct _frame *prefix##_frame;                                          \
    /* True if generator is being executed. */                              \
    char prefix##_running;     /* 运行状态 */                                \
    /* The code object backing the generator */                             \
    PyObject *prefix##_code;   /* 字节码 */                                  \
    /* List of weak reference. */                                           \
    PyObject *prefix##_weakreflist;                                         \
    /* Name of the generator. */                                            \
    PyObject *prefix##_name;                                                \
    /* Qualified name of the generator. */                                  \
    PyObject *prefix##_qualname;

typedef struct {
    /* The gi_ prefix is intended to remind of generator-iterator. */
    _PyGenObject_HEAD(gi)
} PyGenObject;
```

`_frame (PyFrameObject)` 就是生成器的上下文 , `Python` 在执行时实际上是一条 `PyFrameObject` 链 , 每个 `PyFrameObject` 对象中都记录了上一个栈帧对象、字节码对象、字节码执行位置位置

`PyGenObject` 对象对 `PyFrameObject` 做了进一层的封装 , 这是由于生成器的特殊性 , 因为 `PyFrameObject` 对象实际上是一次性的 , 所以必须由其它对象也就是 `PyGenObject` 来保证生成器的正常运行

关于 `Python` 中对 x86 平台栈帧的模拟我们不过多的说明 , 可自行阅读 `《Python源码剖析：深度探索动态语言核心技术》` 一书

## send

在 `Python 2.5, PEP 342` 中 , 添加了将数据发送回暂停的生成器中的功能 , 也就是 `send` 

```C
static PyObject *
gen_send_ex(PyGenObject *gen, PyObject *arg, int exc, int closing)
{
    /* 获取当前的线程环境 */
    PyThreadState *tstate = PyThreadState_GET();
    /* 照当当前生成器的 PyFrameObject 对象 */
    PyFrameObject *f = gen->gi_frame;
    PyObject *result;

    ......

    if (f->f_lasti == -1) {
        /* 未激活 */
        if (arg && arg != Py_None) {
            char *msg = "can't send non-None value to a "
                        "just-started generator";
            if (PyCoro_CheckExact(gen)) {
                msg = NON_INIT_CORO_MSG;
            }
            else if (PyAsyncGen_CheckExact(gen)) {
                msg = "can't send non-None value to a "
                      "just-started async generator";
            }
            PyErr_SetString(PyExc_TypeError, msg);
            return NULL;
        }
    } else {
        /* Push arg onto the frame's value stack */
        result = arg ? arg : Py_None;
        Py_INCREF(result);  /* 如果有参数, 就将其压入栈中 */
        *(f->f_stacktop++) = result;
    }

    /* Generators always return to their most recent caller, not
     * necessarily their creator. */
    Py_XINCREF(tstate->frame);
    assert(f->f_back == NULL);
    f->f_back = tstate->frame;

    gen->gi_running = 1; /* 将生成器设置为运行状态 */
    result = PyEval_EvalFrameEx(f, exc); /* 运行生成器 */
    gen->gi_running = 0;

    /* Don't keep the reference to f_back any longer than necessary.  It
     * may keep a chain of frames alive or it could create a reference
     * cycle. */
    assert(f->f_back == tstate->frame);
    Py_CLEAR(f->f_back);

    /* If the generator just returned (as opposed to yielding), signal
     * that the generator is exhausted. */

    ......

    if (!result || f->f_stacktop == NULL) {
        /* generator can't be rerun, so release the frame */
        /* first clean reference cycle through stored exception traceback */
        PyObject *t, *v, *tb;
        t = f->f_exc_type;
        v = f->f_exc_value;
        tb = f->f_exc_traceback;
        f->f_exc_type = NULL;
        f->f_exc_value = NULL;
        f->f_exc_traceback = NULL;
        Py_XDECREF(t);
        Py_XDECREF(v);
        Py_XDECREF(tb);
        gen->gi_frame->f_gen = NULL;
        gen->gi_frame = NULL;
        Py_DECREF(f);
    }

    return result;
}
```

通过 `send` , 将数据回传到暂停的生成器 , 随后将生成器中的栈帧对象挂载到当前线程上 , 执行完毕后再从当前线程上卸载 , 这样就实现了生成器的调用

`send` 的出现使我们可以进一步对生成器进行控制

生成器的另一种调用方式 `next` 实际上就是 `send(None)` 

```C
static PyObject *
gen_iternext(PyGenObject *gen)
{
    return gen_send_ex(gen, NULL, 0, 0);
}
```

## yield from

在 `Python 3.3, PEP 380` , 增加了 `yield from` , 让你可以以一种干净的方式重构生成器 , 或者说构造生成器链 

```python
def lazy_range(up_to):
    """Generator to return the sequence of integers from 0 to up_to, exclusive."""
    index = 0
    def gratuitous_refactor():
        nonlocal index
        while index < up_to:
            yield index
            index += 1
    yield from gratuitous_refactor()
```

生成器链

```python
def bottom():
    # Returning the yield lets the value that goes up the call stack to come right back
    # down.
    return (yield 42)

def middle():
    return (yield from bottom())

def top():
    return (yield from middle())

# Get the generator.
gen = top()
value = next(gen)
print(value)  # Prints '42'.
try:
    value = gen.send(value * 2)
except StopIteration as exc:
    value = exc.value
print(value)  # Prints '84'.
```

有了生成器的底子 , 我们就可以展开协程篇章了
