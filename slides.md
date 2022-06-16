----
# разработка под tarantool на rust












https://crates.io/crates/tarantool




















---

# зачем?

`        tarantool` + `luajit` == 🤍



```bash
$ find src | grep -o "\.\w*$" | sort | uniq -c | sort -n | tail -4

    36 .cc
    42 .lua
   259 .c
   284 .h
```




















---

# luajit - простой

```lua
                -- keywords

 and       break     do        else      elseif    end
 false     for       function  goto      if        in
 local     nil       not       or        repeat    return
 then      true      until     while
 ```
 ```lua

                -- types

 boolean   number    string    function  table     nil
 thread    userdata  cdata
```




---

# luajit - мощный                              (1/3)

```lua
                -- metatables

t = { foo = 42 }

setmetatable(t, {
    __index = function(self, i)
        error(('key error: %s'):format(i))
    end
})

t.foo -- 42
t.bar -- error: 'key error: bar'
```




---

# luajit - мощный                              (2/3)

```lua
                -- ffi

ffi = require 'ffi'
ffi.cdef [[
    struct vec { int x, y; };
    int vec_norm(struct vec);
]]
lib = ffi.load('/path/to/lib.so')
assert(
    lib.vec_norm({ x = 3, y = 4 }) == 5
)
```






---

# luajit - мощный                              (3/3)

```lua
                -- ffi + metatables
VecProto = {}

function VecProto.norm(self)
    return lib.vec_norm(self)
end

vec = { x = 3, y = 4 }

setmetatable(vec, { __index = VecProto })

assert(
    vec:norm() == 5
)
```
















---

# luajit - быстрый


```lua
             A        B        C         D
   C         1.0      2.3      1.7       4.5
   D         1.1      2.4      999.9     999.9
   Java      1.7      2.6      6.8       13.4
   Go        2.3      3.1      21.4      56.1
-> LuaJIT    3.7      2.5      6.2       999.9 <-
```






source: http://attractivechaos.github.io/plb/



---

# luajit - идеальный?


```
            ▄▄▄▄▄▄▄▄
         ▄█▀▀      ▀▀█▄
       ▄█▀▄██▄        ▀█▄
      █▀ ▀  ▄▀    ▄▀▀▀▀ ▀█
     █▀    ███    ▄█▄    ▀█
     █      ▀     ▀█▀     █
     █                    █
     █  ██▄  ▀▀▀▀▄▄       █
     ▀█ █ █   ▄▄▄▄▄      █▀
      ▀█▀ ▀▀▀▀ ▄▄▄▀    ▄█▀
       █      ▀█     ▄█▀
       █▄     ▀█▄▄▄█▀▀
        ▀▀▀▀▀▀▀
```









---

# luajit - нестрогий

```lua
function baz(foo)
    return foo.bar()
end
```

-`foo`- таблица?

-`bar`- функция?

- `.` - индексация таблицы?

- нужно ли обработать ошибку?







---

# luajit - небезопасный

```lua
file = io.open('/foo/bar')
-- кто должен файл закрыть?


mod = ffi.load('module')
mod.foo(4)
-- кто знает что сделала эта функция?


v3 = ffi.new('int[3]')
v3[4] = 42
-- что произошло?
```
















---







`        rust`ftw?!













---

# rust - строгий

```rust
fn baz(foo: Foo) -> Bar {
    foo.bar()
}
```

-`foo`- имеет тип`Foo`

-`bar`- метод типа`(Foo) -> Bar`

- ошибка явно не указан -> нечего обрабатывать













---

# rust - безопасный

```rust
let v: Vec<T>;
let x = &v[0];
drop(v); // cannot move out of `v` because it is borrowed
bar(x)

let v: Vec<T>;
let x = &v[0];
v.push(13); // cannot borrow `v` as mutable
            // because it is also borrowed as immutable
bar(x)
```






---

# rust - user friendly                         (1/4)


- ресурсы:

  -`docs.rust-lang.org/book`- всё что нужно знать начинающему

  -`docs.rust-lang.org/std` - вся стандартная библиотека

  -`crates.io`              - community модули

  -`stackoverflow.com`      - 0.5% вопросов с тэгом`[rust]`














---

# rust - user friendly                         (2/4)

```bash
# установка

$ curl https://sh.rustup.rs | sh
$ rustup update
```












---

# rust - user friendly                         (3/4)

```bash
# жизненный цикл проекта

$ cargo new
$ cargo build
$ cargo test
$ cargo run
$ cargo doc
```











---

# rust - user friendly                         (4/4)


  - поддержка ide (rls: `VsCode`,`vim`)

    - autocomplete

    - go to definition

    - документация











---




`            tarantool`+`rust`== ?
















---

# tarantool нативный код                       (1/3)


  - нативные`хранимые процедуры`
```lua
      box.schema.func.create('foo.bar', {language = 'C'})
      box.func['foo.bar']:call()
```




















---

# tarantool нативный код                       (2/3)


  -`lua`нативные модули
```lua
      foo = require 'foo'
      foo.bar()
```




















---

# tarantool нативный код                       (3/3)


  -`luajit-ffi`нативные динамические библиотеки
```lua
      ffi = require 'ffi'
      ffi.cdef [[ void bar(void); ]]
      foo = ffi.load('/path/to/libfoo.so')
      foo.bar()
```




















---

# demo хранимых процедур

-`rust`библиотека простого key-value store
    -`create`
    -`insert`
    -`get`

- входная точка на`lua`



































---

# demo: проект

создаём `cargo` проект
```bash
$ cargo new --lib tnt-demo

# tnt-demo
# ├── Cargo.toml  <- билд конигурация/зависимости
# ├── src
# │   └── lib.rs  <- сорсы
# │
# └── target      <- результаты компиляции
```

























---

# demo: конфигурация              `Cargo.toml`

```toml
[package]
name = "tnt-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
tarantool = { version = "0.6", features = ["schema"] }
thiserror = "1.0"

[lib]
crate-type = ["cdylib"]
```










---

# demo: сорсы                         `lib.rs` (1/4)
```rust
use tarantool::space::{Space, Field};

#[tarantool::proc]
fn create(space_name: String) -> Result<(), MyError> {
    let space = Space::builder(&space_name)
        .field(Field::unsigned("key"))
        .field(Field::string("value"))
        .create()?;

    space.index_builder("primary")
        .part("key")
        .create()?;

    Ok(())
}
```




















---

# demo: сорсы                         `lib.rs` (2/4)
```rust
#[tarantool::proc]
fn insert(space_name: String, values: Vec<(usize, String)>)
    -> Result<(), MyError>
{
    let mut space = Space::find(&space_name)
      .ok_or_else(|| MyError::SpaceNotFound(space_name))?;

    for (key, value) in values {
        space.insert(&(key, value))?;
    }
    Ok(())
}
```




















---

# demo: сорсы                         `lib.rs` (3/4)
```rust
use tarantool::index::IteratorType::Eq;

#[tarantool::proc]
fn get(space_name: String, key: usize)
    -> Result<String, MyError>
{
    let space = Space::find(&space_name)
      .ok_or_else(|| MyError::SpaceNotFound(space_name))?;

    if let Some(row) = space.select(Eq, &[key])?.next() {
        let (_, val) = row.into_struct::<(usize, String)>()?;
        Ok(val)
    } else {
        Err(MyError::KeyNotFound(key))
    }
}
```






















---

# demo: сорсы                         `lib.rs` (4/4)
```rust
#[derive(Debug, thiserror::Error)]
enum MyError {
    #[error("space '{0}' not found")]
    SpaceNotFound(String),

    #[error("key '{0}' not found")]
    KeyNotFound(usize),

    #[error("tarantool error: {0}")]
    Tarantool(#[from] tarantool::error::Error),
}
```



















---

# demo: входная точка               `demo.lua`

```lua
box.cfg {}

box.schema.func.create('tnt-demo.create', {language = 'C'})
box.schema.func.create('tnt-demo.insert', {language = 'C'})
box.schema.func.create('tnt-demo.get',    {language = 'C'})

space_name = 'demo_space'
box.func['tnt-demo.create']:call { space_name }
box.func['tnt-demo.insert']:call { space_name, {{1, 'foo'},
                                                {2, 'bar'}}}
assert(box.func['tnt_demo.get']:call{space_name, 1} == 'foo')
assert(box.func['tnt_demo.get']:call{space_name, 2} == 'bar')

print('👌')

os.exit(0)
```




















---

# demo: запускаем

```bash
$ cargo build
$ export LUA_CPATH=target/debug/lib?.so
$ tarantool demo.lua

👌
```




































---

# что может tarantool-module?

- box api: space, index, sequence
- fibers: fiber, cond, channel
- transactions
- logging

- будет ещё

































---


что если на `lua` есть значительная код база?


всё переписывать с нуля?


использовать сишный `api`?























---

# tlua

https://docs.rs/tlua

```rust
let lua = tlua::Lua::new();
lua.exec("
    print('rust + lua == 🤍')
")?;
```

































---

# tlua

если код выполняется один раз, используем `eval`/`exec`
```rust
let lua = tarantool::lua_state();
lua.exec("
    box.cfg { listen = 3301 }
    box.schema.space.create('foo', ..)
    box.space.foo.create_index(..)
")?;

let memory: usize = lua.eval("return box.cfg.memtx_memory")?;
```

















---

# tlua

переиспользуемый код
```rust
use tarantool::tlua::*;
let lua = tarantool::lua_state();

// just eval
let res: Res = lua.eval("return require('foo').bar(42, 'hello')")?;

// no eval
let require: Callable<_> = lua.get("require").unwrap();
let foo: Indexable<_> = require.call_with("foo")?;
let bar: Callable<_> = foo.get("bar")
    .ok_or(MyError::FunctionNotFound("bar"))?;
let res: Res = bar.call_with((42, "hello"))?;

// mixed
let foo_bar: Callable<_> = lua.eval("return require('foo').bar")?;
let res: Res = foo_bar.call_with((42, "hello"))?;
```


















---

# tlua-derive

```rust
use tarantool::tlua;

#[derive(tlua::LuaRead)]
struct Out {
    key: usize,
    value: String,
}

let out: Out = lua.eval("return {key=42, value='hello'}")?;
assert_eq!(out.key, 42);
assert_eq!(out.value, "hello");
```


















---

# tlua-derive

```rust
use tarantool::tlua;

#[derive(tlua::Push)]
struct In {
    key: usize,
    value: String,
}

lua.set("global", In { key: 42, value: "hello".into() })?;
let res: (usize, String) = lua.eval("
    return global.key, global.value
")?;
assert_eq!(res.0, 42);
assert_eq!(res.1, "hello");
```


















---



`   tarantool`+`rust`== 🤍

                        docs.rs/tarantool


`        rust`+`lua` == 🤍

                        docs.rs/tlua


`   tarantool`+`rust`+`lua`== 🤍 + 🤍/2














---

# ссылки




- telegram
           `t.me/picodataru`

- сорсы
        `github.com/picodata/tarantool-module`























---
