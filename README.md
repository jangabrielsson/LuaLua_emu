# LuaLua emulator

Lua implemented in vanilla Lua without any libraries.

Compiles Lua src to a bytecode run by an emulator implmented in Lua. Used in sandboxed environment where loadstring, metatables, and coroutines are not allowed.
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
function debugC()  debug={struct=false,codel=false,trace=true} setTimeout(main,0) end
function debugCStar()  debug={struct=false,codel=true,trace=true} setTimeout(main,0) end

comp = Toolbox_Module.LuaCompiler.inited
function code(str)
  return comp.load(str,nil,nil,comp.stdFuns,debug)()
end
run()
