# LuaLua emulator

Lua implemented in vanilla Lua without any libraries.

Compiles Lua src to a bytecode run by an emulator implemented in Lua. Used in sandboxed environment where loadstring, metatables, and coroutines are not allowed.
~10-30x slower than "native" Lua. Goto and label not implemented but metatables and coroutines works.

Divided up in a parser and a compiler. The parser outputs a parse tree and the compiler creates a nested Lua function of the parse tree. The parser can be parsed by the parser and the parse-tree then compiled by the compiler thus reducing the code foot print.
 
```
dofile("luaParser.lua")
dofile("luaCompiler.lua")
dofile("lib/json.lua")

Toolbox_Module.LuaParser.init(nil,{})
Toolbox_Module.LuaCompiler.init(nil,{})

local function main()
  code([[
  print("Coroutine test")
  local function foo() for i=1,5 do print("Foo:"..i) coroutine.yield() end end
  local function bar() for j=6,10 do print("Bar"..j) coroutine.yield() end end
  local co_foo = coroutine.create(foo)
  local co_bar = coroutine.create(bar)
  for k=1,6 do
    coroutine.resume(co_foo)
    coroutine.resume(co_bar)
  end

-------------------------------------------
  print("metatable test")
  function setDefault (t, d)
     local mt = {__index = function () return d end}
     setmetatable(t, mt)
  end
    
  tab = {x=10, y=20}
  print(tab.x, tab.z)     --> 10   nil
  setDefault(tab, 0)
  print(tab.x, tab.z)     --> 10   0

-------------------------------------------
--  print("HTTP test") -- turning async calls into synchronous...
--  local function myMain(co)

--    local function httpRequest(url)
--       net.HTTPClient():request(url,{
--         options={},
--         success =function(res) 
--             local r = json.decode(res.data)
--             coroutine.resume(co,"success",r.datetime) 
--         end,
--         error = function(res) coroutine.resume(co,"error",res) end,
--       })
--       return coroutine.yield()
--    end

--  -- Now we can call httpRequest as a synchronous function that returns the value immediatly - no more callbacks
--  -- Disclaimer, this is not the best/most efficient way to do it but it demonstrates the principle....
--  local code,res = httpRequest("http://worldtimeapi.org/api/ip") -- Note this server sometimes refuses with 'success'
--  print(code,res)
--  local code,res = httpRequest("http://worldtimeapi.org/api/ip")
--  print(code,res)

--  end
--  local co = coroutine.create(myMain)
--  coroutine.resume(co,co)

 ------------------------------------------------
  print("Illegal boundary test")
  function foo() fopp() print("Never here") end
  function bar() print("..on the other side, 'yield' will barf") coroutine.yield() end
  local c = coroutine.create(foo)
  coroutine.resume(c)
  coroutine.resume(c)
]])
end

function fopp() print("Crossing..") bar() end -- Non-LuaLua function to test with

local debug = {}

function run()  debug={struct=false,codel=false,trace=false} main() end
function debugC()  debug={struct=false,codel=false,trace=true} main() end
function debugCStar()  debug={struct=false,codel=true,trace=true} main() end

comp = Toolbox_Module.LuaCompiler.inited
function code(str)
  return comp.load(str,nil,nil,comp.stdFuns,debug)()
end
run()
```

A simple function with all tracing bells turned on
```
debug={struct=true,codel=true,trace=true}
code([[function fact(x) if x == 0 then return 1 else return x*fact(x-1) end end; print(fact(3))]])
```

gives the following output
```
DEBUG: ["block",[["nop",["function","glob","fun",["glob","fact"],["x"],false,["block",[["if",["==",["var","x"],0],["block",[["return1",[1]]]],[],["block",[["return1",[["*",["var","x"],["call",["glob","fact"],[["-",["var","x"],1]]]]]]]]]]]]],["nop",["call",["glob","print"],[["call",["glob","fact"],[3]]]]]]]
DEBUG: PC:  1 ["function",["x"],false,"glob","fun",["glob","fact"],[["var","x"],["push",0],["eq"],["ifnskip3",5],["push",1],["return",1],["pop"],["goto",10],["var","x"],["var","x"],["push",1],["sub"],["glob","fact"],["call",1,["glob","fact"]],["mul"],["return",1],["pop"],["return",0]]]
DEBUG: PC:  2 ["pop"]
DEBUG: PC:  3 ["push",3]
DEBUG: PC:  4 ["glob","fact"]
DEBUG: PC:  5 ["call",1,["glob","fact"]]
DEBUG: PC:  6 ["glob","print"]
DEBUG: PC:  7 ["call",1,["glob","print"]]
DEBUG: PC:  8 ["pop"]
DEBUG: PC:  9 ["return",0]
TRACE: main    PC:001 ST:000 ["function",["x"],false,"glob","fun",["glob","fact"],[["var","x"],["push",0],["e null
TRACE: main    PC:002 ST:001 ["pop"]                        <non-json>
TRACE: main    PC:003 ST:000 ["push",3]                     null
TRACE: main    PC:004 ST:001 ["glob","fact"]                3
TRACE: main    PC:005 ST:002 ["call",1,["glob","fact"]]     <non-json>
TRACE: main    PC:001 ST:000 ["var","x"]                    null
TRACE: main    PC:002 ST:001 ["push",0]                     3
TRACE: main    PC:003 ST:002 ["eq"]                         0
TRACE: main    PC:004 ST:001 ["ifnskip3",5]                 false
TRACE: main    PC:009 ST:000 ["var","x"]                    null
TRACE: main    PC:010 ST:001 ["var","x"]                    3
TRACE: main    PC:011 ST:002 ["push",1]                     3
TRACE: main    PC:012 ST:003 ["sub"]                        1
TRACE: main    PC:013 ST:002 ["glob","fact"]                2
TRACE: main    PC:014 ST:003 ["call",1,["glob","fact"]]     <non-json>
TRACE: main    PC:001 ST:000 ["var","x"]                    null
TRACE: main    PC:002 ST:001 ["push",0]                     2
TRACE: main    PC:003 ST:002 ["eq"]                         0
TRACE: main    PC:004 ST:001 ["ifnskip3",5]                 false
TRACE: main    PC:009 ST:000 ["var","x"]                    null
TRACE: main    PC:010 ST:001 ["var","x"]                    2
TRACE: main    PC:011 ST:002 ["push",1]                     2
TRACE: main    PC:012 ST:003 ["sub"]                        1
TRACE: main    PC:013 ST:002 ["glob","fact"]                1
TRACE: main    PC:014 ST:003 ["call",1,["glob","fact"]]     <non-json>
TRACE: main    PC:001 ST:000 ["var","x"]                    null
TRACE: main    PC:002 ST:001 ["push",0]                     1
TRACE: main    PC:003 ST:002 ["eq"]                         0
TRACE: main    PC:004 ST:001 ["ifnskip3",5]                 false
TRACE: main    PC:009 ST:000 ["var","x"]                    null
TRACE: main    PC:010 ST:001 ["var","x"]                    1
TRACE: main    PC:011 ST:002 ["push",1]                     1
TRACE: main    PC:012 ST:003 ["sub"]                        1
TRACE: main    PC:013 ST:002 ["glob","fact"]                0
TRACE: main    PC:014 ST:003 ["call",1,["glob","fact"]]     <non-json>
TRACE: main    PC:001 ST:000 ["var","x"]                    null
TRACE: main    PC:002 ST:001 ["push",0]                     0
TRACE: main    PC:003 ST:002 ["eq"]                         0
TRACE: main    PC:004 ST:001 ["ifnskip3",5]                 true
TRACE: main    PC:005 ST:000 ["push",1]                     null
TRACE: main    PC:006 ST:001 ["return",1]                   1
TRACE: main    PC:015 ST:002 ["mul"]                        1
TRACE: main    PC:016 ST:001 ["return",1]                   1
TRACE: main    PC:015 ST:002 ["mul"]                        1
TRACE: main    PC:016 ST:001 ["return",1]                   2
TRACE: main    PC:015 ST:002 ["mul"]                        2
TRACE: main    PC:016 ST:001 ["return",1]                   6
TRACE: main    PC:006 ST:001 ["glob","print"]               6
TRACE: main    PC:007 ST:002 ["call",1,["glob","print"]]    <non-json>
6
TRACE: main    PC:008 ST:001 ["pop"]                        null
TRACE: main    PC:009 ST:000 ["return",0]                   null
Program completed in 0.05 seconds (pid: 38158).
```
