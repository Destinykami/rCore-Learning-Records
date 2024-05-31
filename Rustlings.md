---
title: '清华大学操作系统训练营一阶段 Rustlings'
date: 2024/04/17
permalink: /posts/2024/03/Rustlings
excerpt: '清华大学操作系统训练营一阶段--Rustlings的一点记录'
tags:
  - Rust
  - OS
---
### variable 变量
variable6.rs
```rust
fn main() {
    let number = "T-H-R-E-E"; // don't change this line
    println!("Spell a Number : {}", number);
    number = 3; // don't rename this variable
    println!("Number plus two is : {}", number + 2);
}
```
这里涉及到**影子变量**(shadow),这在c语言中是不允许的。 
修改如下，只需对number进行重新定义。  
```rust
fn main() {
    let number = "T-H-R-E-E"; // don't change this line
    println!("Spell a Number : {}", number);
    let number = 3; // don't rename this variable
    println!("Number plus two is : {}", number + 2);
}
```
### if 条件判断
if3.rs  
if和else的返回值应该是相同的类型  
```rust
pub fn animal_habitat(animal: &str) -> &'static str {
    let identifier = if animal == "crab" {
        1
    } else if animal == "gopher" {
        2
    } else if animal == "snake" {
        3
    } else {
        4
    };

    // DO NOT CHANGE THIS STATEMENT BELOW
    let habitat = if identifier == 1 {
        "Beach"
    } else if identifier == 2 {
        "Burrow"
    } else if identifier == 3 {
        "Desert"
    } else {
        "Unknown"
    };

    habitat
}
```
### primitive_types 原始类型
primitive_types5.rs  
Destructure the `cat` tuple so that the println will work.  

```rust
fn main() {
    let cat = ("Furry McFurson", 3.5);
    //let /* your pattern here */ = cat;
    let (name,age) = cat;

    println!("{} is {} years old.", name, age);
}
```

primitive_types6.rs  
访问元组  
```rust
fn indexing_tuple() {
    let numbers = (1, 2, 3);
    // Replace below ??? with the tuple indexing syntax.
    let second = numbers.1;

    assert_eq!(2, second,
        "This is not the 2nd number in the tuple!")
}
```
### vec 向量
vec1.rs

新建向量vec
```rust
fn array_and_vec() -> ([i32; 4], Vec<i32>) {
    let a = [10, 20, 30, 40]; // a plain array
    let v = vec![10,20,30,40]; // TODO: declare your vector here with the macro for vectors

    (a, v)
}
``` 

vec2.rs

两种方式对vec进行修改
```rust
fn vec_loop(mut v: Vec<i32>) -> Vec<i32> {
    for element in v.iter_mut() {
        // TODO: Fill this up so that each element in the Vec `v` is
        // multiplied by 2.
        *element*=2;
    }

    // At this point, `v` should be equal to [4, 8, 12, 16, 20].
    v
}

fn vec_map(v: &Vec<i32>) -> Vec<i32> {
    v.iter().map(|element| {
        // TODO: Do the same thing as above - but instead of mutating the
        // Vec, you can just return the new number!
        element*2
    }).collect()
}
```
### move_semantics 移动语义
move_semantics2.rs

涉及到变量的问题(?) ，和下面一条对应着来看  

```rust
fn main() {
    let vec0 = Vec::new();

    let mut vec1 = fill_vec(vec0.clone());

    println!("{} has length {}, with contents: `{:?}`", "vec0", vec0.len(), vec0);

    vec1.push(88);

    println!("{} has length {}, with contents `{:?}`", "vec1", vec1.len(), vec1);
}

fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    let mut vec = vec;

    vec.push(22);
    vec.push(44);
    vec.push(66);

    vec
}
```

move_semantics3.rs

```rust
fn main() {
    let vec0 = Vec::new();

    let mut vec1 = fill_vec(vec0);

    println!("{} has length {} content `{:?}`", "vec1", vec1.len(), vec1);

    vec1.push(88);

    println!("{} has length {} content `{:?}`", "vec1", vec1.len(), vec1);
}

fn fill_vec(mut vec: Vec<i32>) -> Vec<i32> {
    vec.push(22);
    vec.push(44);
    vec.push(66);

    vec
}
```
move_semantics5.rs

同一时间一个变量只能有一个可变引用

```rust
fn main() {
    let mut x = 100;
    let y = &mut x;
    let z = &mut x;
    *y += 100;
    *z += 1000;
    assert_eq!(x, 1200);
}
//改为：
fn main() {
    let mut x = 100;
    let y = &mut x;
    *y += 100;
    let z = &mut x;
    *z += 1000;
    assert_eq!(x, 1200);
}
```

move_semantics6.rs

所有权

```rust
fn main() {
    let data = "Rust is great!".to_string();

    get_char(&data);

    string_uppercase(data);
}

// Should not take ownership
fn get_char(data: &String) -> char {
    data.chars().last().unwrap()
}

// Should take ownership
fn string_uppercase(mut data: String) {
    data = data.to_uppercase();

    println!("{}", data);
}
```
### struct 结构体
struct2.rs

在原有的结构体的基础上进行新建结构体，只修改部分变量，其余的和原有模版保持不变  
```rust
fn your_order() {
        let order_template = create_order_template();
        // TODO: Create your own order using the update syntax and template above!
        let your_order = Order{ 
            name: String::from("Hacker in Rust"),
            count: 1,
            ..order_template
        };
        assert_eq!(your_order.name, "Hacker in Rust");
        assert_eq!(your_order.year, order_template.year);
        assert_eq!(your_order.made_by_phone, order_template.made_by_phone);
        assert_eq!(your_order.made_by_mobile, order_template.made_by_mobile);
        assert_eq!(your_order.made_by_email, order_template.made_by_email);
        assert_eq!(your_order.item_number, order_template.item_number);
        assert_eq!(your_order.count, 1);
    }
```
### hashmap 哈希表
hashmap2.rs

Hash maps have a special API for this called entry that takes the key you want to check as a parameter. The return value of the entry method is an enum called Entry that represents a value that might or might not exist. Let’s say we want to check whether the key for the Yellow team has a value associated with it. 

The or_insert method on Entry is defined to return a mutable reference to the value for the corresponding Entry key if that key exists, and if not, inserts the parameter as the new value for this key and returns a mutable reference to the new value. This technique is much cleaner than writing the logic ourselves and, in addition, plays more nicely with the borrow checker.

```rust
    for fruit in fruit_kinds {
        // TODO: Insert new fruits if they are not already present in the
        // basket. Note that you are not allowed to put any type of fruit that's
        // already present!
        basket.entry(fruit).or_insert(1);
    }
```
### error 错误处理
errors2.rs  

? 是 Rust 中的错误处理操作符。通常用于尝试解析或执行可能失败的操作，并在出现错误时提前返回错误，以避免程序崩溃或出现未处理的错误。

具体来说，? 用于处理 Result 或 Option 类型的返回值。
```rust
pub fn total_cost(item_quantity: &str) -> Result<i32, ParseIntError> {
    let processing_fee = 1;
    let cost_per_item = 5;
    let qty = item_quantity.parse::<i32>() ?; //加个问号就好

    Ok(qty * cost_per_item + processing_fee)
}
```

### Traits 特征

A trait is a collection of methods.

Data types can implement traits. To do so, the methods making up the trait are defined for the data type. For example, the `String` data type implements the `From<&str>` trait. This allows a user to write `String::from("hello")`.

In this way, traits are somewhat similar to Java interfaces and C++ abstract classes.

Some additional common Rust traits include:

- `Clone` (the `clone` method)
- `Display` (which allows formatted display via `{}`)
- `Debug` (which allows formatted display via `{:?}`)

Because traits indicate shared behavior between data types, they are useful when writing generics.

### Further information

- [Traits](https://doc.rust-lang.org/book/ch10-02-traits.html)

traits4.rs:

创建关于共享行为(trait)的两个实例的泛型  
```rust
// YOU MAY ONLY CHANGE THE NEXT LINE
fn compare_license_types<T:Licensed,U:Licensed>(software: T, software_two: U) -> bool {
    software.licensing_info() == software_two.licensing_info()
}
```
traits5.rs:  
使用+来放行实现了多种共享行为的类型  
```rust
fn some_func<T:SomeTrait+OtherTrait>(item: T) -> bool {
    item.some_function() && item.other_function()
}
```
### iterators 迭代器
iterators1.rs:

迭代器的使用，值的不可变引用  
```rust
fn main() {
    let my_fav_fruits = vec!["banana", "custard apple", "avocado", "peach", "raspberry"];

    let mut my_iterable_fav_fruits = my_fav_fruits.iter();   // TODO: Step 1

    assert_eq!(my_iterable_fav_fruits.next(), Some(&"banana"));
    assert_eq!(my_iterable_fav_fruits.next(), Some(&"custard apple"));     // TODO: Step 2
    assert_eq!(my_iterable_fav_fruits.next(), Some(&"avocado"));
    assert_eq!(my_iterable_fav_fruits.next(), Some(&"peach"));     // TODO: Step 3
    assert_eq!(my_iterable_fav_fruits.next(), Some(&"raspberry"));
    assert_eq!(my_iterable_fav_fruits.next(), None);     // TODO: Step 4
}
```
iterators3.rs:  

```rust
// Calculate `a` divided by `b` if `a` is evenly divisible by `b`.
// Otherwise, return a suitable error.
pub fn divide(a: i32, b: i32) -> Result<i32, DivisionError> {
    if b == 0 {
        return Err(DivisionError::DivideByZero);
    } else if a % b == 0 {
        Ok(a / b)
    } else {
        return Err(DivisionError::NotDivisible(NotDivisibleError {
            dividend: a,
            divisor: b,
        }));
    }
}

// Complete the function and return a value of the correct type so the test
// passes.
// Desired output: Ok([1, 11, 1426, 3])
fn result_with_list() -> Result<Vec<i32>, DivisionError> {
    let numbers = vec![27, 297, 38502, 81];
    //let division_results = numbers.into_iter().map(|n| divide(n, 27));
    let mut buf = Vec::<i32>::new();
    for n in numbers {
        match divide(n, 27) {
            Ok(r) => buf.push(r),
            Err(e) => return Err(e),
        }
    }

    Ok(buf)
}

// Complete the function and return a value of the correct type so the test
// passes.
// Desired output: [Ok(1), Ok(11), Ok(1426), Ok(3)]
fn list_of_results() -> Vec<Result<i32, DivisionError>> {
    let numbers = vec![27, 297, 38502, 81];
    //let division_results = numbers.into_iter().map(|n| divide(n, 27));
    let mut buf = Vec::new();

    for n in numbers {
        match divide(n, 27) {
            Ok(r) => buf.push(Ok(r)),
            Err(e) => buf.push(Err(e)),
        }
    }
    buf
}
```

iterators4.rs:  

阶乘居然还能这样算...

```rust
pub fn factorial(num: u64) -> u64 {
    // Complete this function to return the factorial of num
    // Do not use:
    // - return
    // Try not to use:
    // - imperative style loops (for, while)
    // - additional variables
    // For an extra challenge, don't use:
    // - recursion
    // Execute `rustlings hint iterators4` for hints.
    (1..=num).product()
}
```
iterator5.rs: 

* iter()获取map的迭代器
* filter()HashMap中第一个元素value相等的值
* count()用来返回数量

```rust
fn count_iterator(map: &HashMap<String, Progress>, value: Progress) -> usize {
    // map is a hashmap with String keys and Progress values.
    // map = { "variables1": Complete, "from_str": None, ... }
    //todo!();
    let count=map
        .iter()
        .filter(|v| *v.1==value)
        .count();
    count
}
fn count_collection_iterator(collection: &[HashMap<String, Progress>], value: Progress) -> usize {
    // collection is a slice of hashmaps.
    // collection = [{ "variables1": Complete, "from_str": None, ... },
    //     { "variables2": Complete, ... }, ... ]
    let mut my_map1 = collection[0].clone();
    let mut my_map2 = collection[1].clone();
	// 调用extend方法，将my_map1转换为一个迭代器，并将其键值对添加到my_map2中
    my_map2.extend(my_map1.into_iter());
	// 与上一个相同
    let count11 = my_map2
        .iter()
        .filter(|v| *v.1 == value)
        .count();
    count11
}
```

### smart_pointers  智能指针
box1.rs  
在编译时，Rust 需要知道一个类型占用了多少空间。 这对于递归类型来说是有问题的，因为一个值可以具有相同类型的另一个值作为其自身的一部分。 为了解决这个问题，我们可以使用“Box”-- 一个用于在堆上存储数据的智能指针，它还允许我们包装递归类型。  
```rust
pub enum List {
    Cons(i32, Box<List>),
    Nil,
}

fn main() {
    println!("This is an empty cons list: {:?}", create_empty_list());
    println!(
        "This is a non-empty cons list: {:?}",
        create_non_empty_list()
    );
}

pub fn create_empty_list() -> List {
    List::Nil
}

pub fn create_non_empty_list() -> List {
    List::Cons(0, Box::new(List::Nil))
}
```

rc1.rs

Box 自定义了 Drop 用来释放 box 所指向的堆空间。
Rc 记录了堆上数据的引用数量以便可以拥有多个所有者

arc1.rs

Given a Vec of u32 called "numbers" with values ranging from 0 to 99 -- [ 0, 1, 2, ..., 98, 99 ] We would like to use this set of numbers within 8 different threads simultaneously. Each thread is going to get the sum of every eighth value, with an offset.

**Because we are using threads, our values need to be thread-safe. Therefore, we are using Arc.**

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let numbers: Vec<_> = (0..100u32).collect();
    let shared_numbers = Arc::new(numbers);// TODO
    let mut joinhandles = Vec::new();

    for offset in 0..8 {
        let child_numbers = Arc::clone(&shared_numbers);// TODO
        joinhandles.push(thread::spawn(move || {
            let sum: u32 = child_numbers.iter().filter(|&&n| n % 8 == offset).sum();
            println!("Sum of offset {} is {}", offset, sum);
        }));
    }
    for handle in joinhandles.into_iter() {
        handle.join().unwrap();
    }
}

```
cow1.rs

Cow **(Clone-On-Write)** 是一个写时克隆智能指针。 它可以封装并提供对借用数据的不可变访问，并在需要突变或所有权时延迟克隆数据。 该类型旨在通过 Borrow 特征处理一般借用数据。Cow 类型可以用来避免不必要的内存分配和复制操作，从而提高程序的性能和效率

作用:  

* 读写分离：在一些业务场景中，需要对某个数据结构进行多次读取和少量修改，但是每次修改都会导致内存分配和复制操作，从而影响程序的性能和效率。Cow 类型可以通过克隆操作来避免这个问题，从而提高程序的性能和效率。

* 借用检查：在 Rust 中，借用检查是一项重要的安全特性，可以避免程序中出现内存安全问题。但是，在某些情况下，借用检查会导致代码的复杂度和可读性变差。Cow 类型可以通过引用和克隆操作来解决这个问题，从而简化代码的实现和维护。

```rust
fn reference_mutation() -> Result<(), &'static str> {
        // Clone occurs because `input` needs to be mutated.
        let slice = [-1, 0, 1];
        let mut input = Cow::from(&slice[..]);
        match abs_all(&mut input) {
            Cow::Owned(_) => Ok(()),
            _ => Err("Expected owned value"),
        }
    }

    #[test]
    fn reference_no_mutation() -> Result<(), &'static str> {
        // No clone occurs because `input` doesn't need to be mutated.
        let slice = [0, 1, 2];
        let mut input = Cow::from(&slice[..]);
        match abs_all(&mut input) {
            // TODO
            Cow::Borrowed(_) => Ok(()),
            _ => Err("Expected borrowed value"),
        }
    }

    #[test]
    fn owned_no_mutation() -> Result<(), &'static str> {
        // We can also pass `slice` without `&` so Cow owns it directly. In this
        // case no mutation occurs and thus also no clone, but the result is
        // still owned because it was never borrowed or mutated.
        let slice = vec![0, 1, 2];
        let mut input = Cow::from(slice);
        match abs_all(&mut input) {
            // TODO
            Cow::Owned(_) => Ok(()),
            _ => Err("Expected owned value"),
        }
    }

    #[test]
    fn owned_mutation() -> Result<(), &'static str> {
        // Of course this is also the case if a mutation does occur. In this
        // case the call to `to_mut()` returns a reference to the same data as
        // before.
        let slice = vec![-1, 0, 1];
        let mut input = Cow::from(slice);
        match abs_all(&mut input) {
            // TODO
            Cow::Owned(_) => Ok(()),
            _ => Err("Expected owned value"),
        }
    }
```

### thread 线程
thread1.rs

```rust
fn main() {
    let mut handles = vec![];
    for i in 0..10 {
        handles.push(thread::spawn(move || {
            let start = Instant::now();
            thread::sleep(Duration::from_millis(250));
            println!("thread {} is complete", i);
            start.elapsed().as_millis()
        }));
    }

    let mut results: Vec<u128> = vec![];
    for handle in handles {
        // TODO: a struct is returned from thread::spawn, can you use it?
        results.push(handle.join().unwrap());
    }

    if results.len() != 10 {
        panic!("Oh no! All the spawned threads did not finish!");
    }

    println!();
    for (i, result) in results.into_iter().enumerate() {
        println!("thread {} took {}ms", i, result);
    }
}
```
thread2.rs

访问全局变量时，需要用到互斥锁来保证线程安全。  

```rust
use std::sync::Arc;
use std::sync::Mutex;
use std::thread;
use std::time::Duration;

struct JobStatus {
    jobs_completed: u32,
}

fn main() {
    let status = Arc::new(Mutex::new(JobStatus { jobs_completed: 0 }));
    let mut handles = vec![];
    for _ in 0..10 {
        let status_shared = Arc::clone(&status);
        let handle = thread::spawn(move || {
            thread::sleep(Duration::from_millis(250));
            // TODO: You must take an action before you update a shared value
            status_shared.lock().unwrap().jobs_completed+=1; //later
        });
        handles.push(handle);
    }
    for handle in handles {
        handle.join().unwrap();
        // TODO: Print the value of the JobStatus.jobs_completed. Did you notice
        // anything interesting in the output? Do you have to 'join' on all the
        // handles?
        println!("jobs completed {}", status.lock().unwrap().jobs_completed);
    }
}
```

thread3.rs

**multiple producer, single consumer**  
单消费者，多生产者问题  

应该是涉及到所有权的问题，把信号量clone了然后再给线程。

```rust
use std::sync::mpsc;
use std::sync::Arc;
use std::thread;
use std::time::Duration;

struct Queue {
    length: u32,
    first_half: Vec<u32>,
    second_half: Vec<u32>,
}

impl Queue {
    fn new() -> Self {
        Queue {
            length: 10,
            first_half: vec![1, 2, 3, 4, 5],
            second_half: vec![6, 7, 8, 9, 10],
        }
    }
}

fn send_tx(q: Queue, tx: mpsc::Sender<u32>) -> () {
    let qc = Arc::new(q);
    let qc1 = Arc::clone(&qc);
    let qc2 = Arc::clone(&qc);
    let tx1=tx.clone(); //new
    let tx2=tx.clone(); //new
    thread::spawn(move || {
        for val in &qc1.first_half {
            println!("sending {:?}", val);
            tx1.send(*val).unwrap(); //edited
            thread::sleep(Duration::from_secs(1));
        }
    });

    thread::spawn(move || {
        for val in &qc2.second_half {
            println!("sending {:?}", val);
            tx2.send(*val).unwrap(); //edited
            thread::sleep(Duration::from_secs(1));
        }
    });
}

fn main() {
    let (tx, rx) = mpsc::channel();
    let queue = Queue::new();
    let queue_length = queue.length;

    send_tx(queue, tx);

    let mut total_received: u32 = 0;
    for received in rx {
        println!("Got: {}", received);
        total_received += 1;
    }

    println!("total numbers received: {}", total_received);
    assert_eq!(total_received, queue_length)
}
```

### macros 宏

When you call a macro, you need to add something special compared to a regular function call.

macros1.rs

```rust
macro_rules! my_macro {
    () => {
        println!("Check out my macro!");
    };
}

fn main() {
    //my_macro();
    my_macro!();//加了个感叹号
}

```
macros2.rs

宏要先定义才能使用，移动顺序即可

macros3.rs

为了在其模块之外使用宏，需要做一些事情，对于模块来说是特殊的，用于将宏提取到其父级中。

`#[macro_export]`
```rust
mod macros {
    #[macro_export]
    macro_rules! my_macro {
        () => {
            println!("Check out my macro!");
        };
    }
}

fn main() {
    my_macro!();
}
```

macros4.rs

分号分隔

### clippy
Rust 在标准库中存储了长精度或无限精度数学常数的最高精度版本。我们可能会想对某些数学常数使用我们自己的近似值，但 Clippy 认为这些不精确的数学常数是潜在错误的来源。  

clippy2.rs

`for` loops over Option values are more clearly expressed as an `if let`

```rust
fn main() {
    let mut res = 42;
    let option = Some(12);
    // for x in option {
    //     res += x;
    // }
    if let Some(x)=option{
        res+=x;
    }
    println!("{}", res);
}
```

### Type conversions 类型转换
Rust offers a multitude of ways to convert a value of a given type into another type.

The simplest form of type conversion is a type cast expression. It is denoted with the binary operator `as`. For instance, `println!("{}", 1 + 1.0);` would not compile, since `1` is an integer while `1.0` is a float. However, `println!("{}", 1 as f32 + 1.0)` should compile.  

Rust also offers traits that facilitate type conversions upon implementation. These traits can be found under the [`convert`](https://doc.rust-lang.org/std/convert/index.html) module.
The traits are the following:

- `From` and `Into` covered in [`from_into`](from_into.rs)
- `TryFrom` and `TryInto` covered in [`try_from_into`](try_from_into.rs)
- `AsRef` and `AsMut` covered in [`as_ref_mut`](as_ref_mut.rs)

Furthermore, the `std::str` module offers a trait called [`FromStr`](https://doc.rust-lang.org/std/str/trait.FromStr.html) which helps with converting strings into target types via the `parse` method on strings. If properly implemented for a given type `Person`, then `let p: Person = "Mark,20".parse().unwrap()` should both compile and run without panicking.

These should be the main ways ***within the standard library*** to convert data into your desired types.

from_into.rs

输入str，转化为Person对象。 用split分割

```rust
impl From<&str> for Person {
    fn from(s: &str) -> Person {
        if (!s.is_empty()) {
            let mut splited = s.split(",");
            if (splited.clone().count() == 2) {
                let name = splited.next().unwrap().to_string();
                if (!name.is_empty()) {
                    if let Ok(var_name) = splited.next().unwrap().parse::<usize>() {
                        return Person {
                            name: name,
                            age: var_name,
                        };
                    }
                }
            }
        }

        Person::default()
    }
}
```
from_str.rs

和上面的挺像的...就是加了错误处理看起来  
```rust
impl FromStr for Person {
    type Err = ParsePersonError;
    fn from_str(s: &str) -> Result<Person, Self::Err> {
        if !s.is_empty() {
            let mut splited = s.split(",");
            if splited.clone().count() == 2 {
                let name = splited.next().unwrap().to_string();
                if !name.is_empty() {
                    match splited.next().unwrap().parse::<usize>() {
                        Ok(age) => Ok(Person {
                            name: name,
                            age: age,
                        }),
                        Err(err) => Err(ParsePersonError::ParseInt(err)),
                    }
                } else {
                    Err(ParsePersonError::NoName)
                }
            } else {
                Err(ParsePersonError::BadLen)
            }
        } else {
            Err(ParsePersonError::Empty)
        }
    }
}
```

try_from_into.rs

```rust
impl TryFrom<(i16, i16, i16)> for Color {
    type Error = IntoColorError;
    fn try_from(tuple: (i16, i16, i16)) -> Result<Self, Self::Error> {
        let range = 0..=255;
        match tuple {
            (r, g, b) if range.contains(&r) && range.contains(&g) && range.contains(&b) => {
                Ok(Color {
                    red: r as u8,
                    green: g as u8,
                    blue: b as u8,
                })
            }
            _ => Err(IntoColorError::IntConversion),
        }
    }
}

// Array implementation
impl TryFrom<[i16; 3]> for Color {
    type Error = IntoColorError;
    fn try_from(arr: [i16; 3]) -> Result<Self, Self::Error> {
        let range = 0..=255;
        match arr {
            [r, g, b] if range.contains(&r) && range.contains(&g) && range.contains(&b) => {
                Ok(Color {
                    red: r as u8,
                    green: g as u8,
                    blue: b as u8,
                })
            }
            _ => Err(IntoColorError::IntConversion),
        }
    }
}

// Slice implementation
impl TryFrom<&[i16]> for Color {
    type Error = IntoColorError;
    fn try_from(slice: &[i16]) -> Result<Self, Self::Error> {
        if slice.len() != 3 {
            return Err(IntoColorError::BadLen);
        }
        let range = 0..=255;
        match slice {
            [r, g, b] if range.contains(r) && range.contains(g) && range.contains(b) => Ok(Color {
                red: *r as u8,
                green: *g as u8,
                blue: *b as u8,
            }),
            _ => Err(IntoColorError::IntConversion),
        }
    }
}
```

as_ref_mut.rs  

不大懂呢  

### unsafe
An `unsafe` in Rust serves as a contract.

When `unsafe` is marked on a code block enclosed by curly braces, it declares an observance of some contract, such as the validity of somepointer parameter, the ownership of some memory address. However, like the text above, you still need to state how the contract is observed in the comment on the code block.

也就是说对裸指针的引用需要用unsafe包裹起来

test5.rs

```rust
unsafe fn modify_by_address(address: usize) {
    // TODO: Fill your safety notice of the code block below to match your
    // code's behavior and the contract of this function. You may use the
    // comment of the test below as your format reference.
    unsafe {
        let ptr = address as *mut u32;
        unsafe { *ptr = 0xAABBCCDD as u32 }
    }
}
```

test6.rs

```rust
unsafe fn raw_pointer_to_box(ptr: *mut Foo) -> Box<Foo> {
    // SAFETY: The `ptr` contains an owned box of `Foo` by contract. We
    // simply reconstruct the box from that pointer.
    let mut ret: Box<Foo> = unsafe { Box::from_raw(ptr) };
    ret.b = Some("hello".to_string());
    ret
}
```

### Some Addition

test7,test8中对于cargo的build.rs的一些说明 

test9中关于extern，引入外部abi接口

Rust is highly capable of sharing FFI interfaces with C/C++ and other statically compiled
languages, and it can even link within the code itself! It makes it through the extern
block, just like the code below.

The short string after the `extern` keyword indicates which ABI the externally imported
function would follow. In this exercise, "Rust" is used, while other variants exists like
"C" for standard C ABI, "stdcall" for the Windows ABI.

The externally imported functions are declared in the extern blocks, with a semicolon to
mark the end of signature instead of curly braces. Some attributes can be applied to those
function declarations to modify the linking behavior, such as #[link_name = ".."] to
modify the actual symbol names.

If you want to export your symbol to the linking environment, the `extern` keyword can
also be marked before a function definition with the same ABI string note. The default ABI
for Rust functions is literally "Rust", so if you want to link against pure Rust functions,
the whole extern term can be omitted.

Rust mangles symbols by default, just like C++ does. To suppress this behavior and make
those functions addressable by name, the attribute #[no_mangle] can be applied.

In this exercise, your task is to make the testcase able to call the `my_demo_function` in
module Foo. the `my_demo_function_alias` is an alias for `my_demo_function`, so the two
line of code in the testcase should call the same function.

```rust
extern "Rust" {
    fn my_demo_function(a: u32) -> u32;
    #[link_name = "my_demo_function"]
    fn my_demo_function_alias(a: u32) -> u32;
}

mod Foo {
    // No `extern` equals `extern "Rust"`.
    #[no_mangle]
    fn my_demo_function(a: u32) -> u32 {
        a
    }
}
```

### algorithm 
algorithm1.rs  
single linked list merge

```rust

```

algorithm2.rs
```rust

```

algorithm3.rs
```rust

```

algorithm4.rs
```rust

```

algorithm5.rs
```rust

```

algorithm6.rs
```rust

```

algorithm7.rs
```rust

```

algorithm8.rs
```rust

```

algorithm9.rs
```rust

```
