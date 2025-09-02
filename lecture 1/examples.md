### Python: динамическая типизация и аннотации не гарантируют проверку типов
```python
# Python
def add(a: int, b: int) -> int:
    return a + b

print(add(1, 2))        # 3
print(add("1", "2"))    # '12' — аннотации не помешали! Проверки нет в рантайме
```

Output:
```
3
12
```


### Python: GIL и многопоточность (CPU-bound не ускоряется потоками, но ускоряется процессами)
```python
# Python
import time, threading, multiprocessing

def cpu_bound(n):
    s = 0
    for i in range(10_000_00):
        s += (i * i) % n
    return s

def run_threads():
    t1 = threading.Thread(target=cpu_bound, args=(97,))
    t2 = threading.Thread(target=cpu_bound, args=(97,))
    t0 = time.time()
    t1.start(); t2.start(); t1.join(); t2.join()
    print("Threads:", round(time.time() - t0, 2), "s")

def run_processes():
    t0 = time.time()
    with multiprocessing.Pool(2) as p:
        p.map(cpu_bound, [97, 97])
    print("Processes:", round(time.time() - t0, 2), "s")

if __name__ == "__main__":
    run_threads()     # обычно ≈ время одной задачи из‑за GIL
    run_processes()   # обычно быстрее на CPU‑bound
```

Output:
```
Threads: 9.51 s
Processes: 5.12 s
```


### Python: asyncio — реактивный/асинхронный стиль против блокирующего
```python
# Python
import asyncio, time

async def fetch(i):
    await asyncio.sleep(1)  # имитация I/O
    return f"done {i}"

async def main():
    t0 = time.time()
    results = await asyncio.gather(*(fetch(i) for i in range(5)))
    print(results, "in", round(time.time() - t0, 2), "s")  # ~1s, а не ~5s

if __name__ == "__main__":
    asyncio.run(main())
```

Output:
```
['done 0', 'done 1', 'done 2', 'done 3', 'done 4'] in 1.0 s
```

### Go: «встроенная реактивность» — горутины и каналы
```textmate
// Go
package main

import (
	"fmt"
	"time"
)

func worker(id int, ch chan<- string) {
	time.Sleep(500 * time.Millisecond)
	ch <- fmt.Sprintf("done %d", id)
}

func main() {
	ch := make(chan string)
	for i := 0; i < 5; i++ {
		go worker(i, ch)
	}
	for i := 0; i < 5; i++ {
		fmt.Println(<-ch)
	}
}
```

### C: ручное управление памятью, пример «выстрела в ногу» (use-after-free)
```textmate
// C
#include <stdio.h>
#include <stdlib.h>

int main() {
    int *p = malloc(sizeof(int));
    *p = 42;
    free(p);
    // Опасно: использование освобожденной памяти — неопределенное поведение
    printf("%d\n", *p); 
    return 0;
}
```

### Reference counting и циклические ссылки в Python
```python
# Python
import gc

class Node:
    def __init__(self, name: str):
        self.name = name
        self.ref = None
    def __del__(self):
        print(f"Finalized {self.name}")

a = Node("a")
b = Node("b")
c = Node("c")

# a и b ссылаются друг на друга, c - ни на кого
a.ref = b
b.ref = a

# Удаляем ссылки на объекты. Нода c уничтожится сразу, т.к. сработает reference counting. Остальные - только после GC.
print("Performing del...")
del a, b, c

# Явно форсируем GC
print("After del, forcing GC...")
unreachable = gc.collect()

print("Unreachable objects:", unreachable)
```

Output:
```
Performing del...
Finalized c
After del, forcing GC...
Finalized a
Finalized b
Unreachable objects: 16
```

### Слабые ссылки для разрыва циклов
```python
# Python
import weakref

class Node:
    def __init__(self):
        self.ref = None

a = Node(); b = Node()
a.ref = weakref.ref(b)  # слабая ссылка
b.ref = a               # обычная
del a, b  # объекты освобождаются без участия цикличного GC
```


### Rust: borrow checker — запрещённая двойная мутабельная ссылка
```textmate
// Rust
fn main() {
    let mut s = String::from("hi");
    let r1 = &mut s;
    // let r2 = &mut s; // ошибка компиляции: нельзя две &mut одновременно
    r1.push_str("!");
    println!("{}", r1);
}
```


### Rust: передача данных между потоками безопасно (Send + 'static), совместное владение через Arc
```textmate
// Rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for _ in 0..4 {
        let c = Arc::clone(&counter);
        let h = thread::spawn(move || {
            for _ in 0..100_000 {
                *c.lock().unwrap() += 1;
            }
        });
        handles.push(h);
    }
    for h in handles { h.join().unwrap(); }

    println!("counter = {}", *counter.lock().unwrap());
}
```


### Микросервисы: минимальный REST (FastAPI)
```python
# Python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    id: int
    name: str

@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/items")
def create_item(item: Item):
    return {"created": item}
```


### Микросервисы: gRPC — определение контракта и сервер (Python)
.proto:
```
// Proto
syntax = "proto3";
package demo;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
```


Сервер:
```python
# Python
import grpc
from concurrent import futures
import demo_pb2, demo_pb2_grpc

class Greeter(demo_pb2_grpc.GreeterServicer):
    def SayHello(self, request, context):
        return demo_pb2.HelloReply(message=f"Hello, {request.name}")

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=4))
    demo_pb2_grpc.add_GreeterServicer_to_server(Greeter(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```


### Java: JIT
```java
// Java
public class HotLoop {
    static long work(int n) {
        long s = 0;
        for (int i = 0; i < 10_000_000; i++) s += i % n;
        return s;
    }
    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            long t = System.nanoTime();
            work(97); // JIT «разогреется», поздние итерации быстрее
            System.out.println((System.nanoTime() - t)/1e6 + " ms");
        }
    }
}
```