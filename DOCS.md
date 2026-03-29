# AXIS Core — Full Documentation

> Framework หลักสำหรับ FiveM/RedM server
> รองรับทั้ง Client และ Server ด้วย API เดียวกัน

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Event System](#event-system)
3. [Player Class](#player-class)
4. [Membership System](#membership-system)
5. [Utility Modules](#utility-modules)
6. [Commands](#commands)
7. [Configuration](#configuration)
8. [Data Sync](#data-sync)
9. [Internal Events](#internal-events)

---

## Getting Started

### ใช้ AXIS ใน resource อื่น

```lua
-- shared/client/server
AXIS = exports["axis_core"]:getSharedObject()
```

### หรือใช้ imports.lua

```lua
-- fxmanifest.lua
shared_scripts { '@axis_core/shared/imports.lua' }
```

เมื่อใส่ imports.lua จะได้ `AXIS` global ทันที พร้อม `OnPlayerData` callback:

```lua
OnPlayerData = function(key, val, last)
    if key == "job" then
        print("Job changed from", last, "to", val)
    end
end
```

---

## Event System

ระบบ Event สองทิศทางพร้อม callback และ timeout

### Server → Client

#### `AXIS:TriggerClientEvent(options)`

ส่ง event จาก Server ไป Client และรอผลลัพธ์กลับ

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | ชื่อ event |
| `source` | number \| table | Yes | Player source หรือ {source1, source2, ...} |
| `args` | table | No | Arguments |
| `timeout` | number | No | Timeout (ms) |
| `callback` | function | No | `function(source, ...)` |

```lua
AXIS:TriggerClientEvent({
    name = "getPosition",
    source = playerId,
    timeout = 5000,
    callback = function(source, x, y, z)
        print(("Player %d is at %.1f, %.1f, %.1f"):format(source, x, y, z))
    end
})
```

### Client → Server

#### `AXIS:TriggerServerEvent(options)`

ส่ง event จาก Client ไป Server และรอผลลัพธ์กลับ

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | ชื่อ event |
| `args` | table | No | Arguments |
| `timeout` | number | No | Timeout (ms) |
| `callback` | function | No | `function(...)` |

```lua
AXIS:TriggerServerEvent({
    name = "getPlayerMoney",
    args = { "bank" },
    timeout = 5000,
    callback = function(money)
        print("Bank balance:", money)
    end
})
```

### Register Handler

#### `AXIS:RegisterNetEvent(name, func)`

ทั้ง Client และ Server ใช้คำสั่งเดียวกัน

```lua
-- Server
AXIS:RegisterNetEvent("getPlayerMoney", function(source, accountType)
    local player = AXIS:GetPlayer(source)
    return player:getAccount(accountType)
end)

-- Client
AXIS:RegisterNetEvent("showNotification", function(message)
    ShowNotification(message)
    return true
end)
```

> ค่าที่ `return` จาก handler จะถูกส่งกลับไปยังฝั่งผู้ส่งอัตโนมัติ

### Event Object Properties

| Property | Type | Description |
|----------|------|-------------|
| `id` | number | Event ID ไม่ซ้ำ |
| `name` | string | ชื่อ event |
| `source` | number/table | Player source (เฉพาะ TriggerClientEvent) |
| `args` | table | Arguments ที่ส่งไป |
| `timeout` | number | Timeout timestamp |
| `callback` | function | Callback (ตั้งภายหลังได้) |
| `responded` | boolean | ตอบกลับแล้วหรือยัง |

---

## Player Class

Player instance ได้มาจาก `AXIS:GetPlayer(source)` (Server เท่านั้น)

### รับ Player

```lua
local player = AXIS:GetPlayer(source)          -- จาก source
local player = AXIS:GetPlayerByIdentifier(id)  -- จาก identifier
local allPlayers = AXIS:GetPlayers()           -- ทุกคน
```

### ค้นหา Player

```lua
-- หาคนที่เป็น police
local police = AXIS:ExtendedPlayers("job", "police")

-- หาหลาย job
local officers = AXIS:ExtendedPlayers("job", {"police", "sheriff"})

-- หาหลาย filter
local result = AXIS:ExtendedPlayers({"firstname", "job"}, {firstname = "Jack", job = "police"})

-- นับจำนวน
local count = AXIS:GetNumPlayers("job", "police")
local total = AXIS:GetNumPlayers()
```

### Account Management

```lua
player:setAccountMoney("bank", 5000)
player:addAccountMoney("cash", 500)
player:removeAccountMoney("bank", 100)

local balance = player:getAccount("bank")
local accounts = player:getAccounts()
```

### Inventory Management

```lua
player:addInventoryItem("bread", 5)
player:removeInventoryItem("bread", 2)
player:setInventoryItem("bread", 10)

local item = player:getInventoryItem("bread")
local canCarry = player:canCarryItem("bread", 5)
local canSwap = player:canSwapItem("bread", 1, "water", 2)

local weight = player:getWeight()
local maxWeight = player:getMaxWeight()
player:setMaxWeight(10000)

local inventory = player:getInventory()
```

### Job & Grade

```lua
player:setJob("police")
player:setGrade(3)

local job = player:getJob()   -- {name, label, grade, ...}
```

### Group

```lua
player:setGroup("admin")
local group = player:getGroup()  -- {name, label, permissions}
```

### Metadata

```lua
-- ตั้งค่า
player:setMeta("health", 200)
player:setMeta("skills", { strength = 10, agility = 5 })
player:setMeta("xp", 100, "mining")  -- subIndex

-- อ่านค่า
local health = player:getMeta("health")
local skills = player:getMeta("skills")
local miningXp = player:getMeta("skills", "mining")  -- subIndex

-- หลาย subIndex
local data = player:getMeta("skills", {"strength", "agility"})
-- return { strength = 10, agility = 5 }

-- อ่านทั้งหมด
local allMeta = player:getMeta()

-- ลบ
player:setMeta("tempBuff", nil)
```

### Personal Info

```lua
player:setFirstName("John")
player:setLastName("Doe")
player:setDOB("01/01/2000")
player:setSex("m")
player:setHeight(180)

local name = player:getFullName()   -- "John Doe"
local first = player:getFirstName() -- "John"
local last = player:getLastName()   -- "Doe"
local dob = player:getDOB()         -- "01/01/2000"
local sex = player:getSex()         -- "m"
local height = player:getHeight()   -- 180
```

### Weapon Management

```lua
-- เพิ่มอาวุธ
player:addWeapon("WEAPON_PISTOL", {ammo = {AMMO_PISTOL = 50}}, nil, {component_slot = "component_value"})

-- ลบอาวุธ
player:removeWeapon(weaponIdentifier)

-- กระสุน
player:addWeaponAmmo(weaponIdentifier, "AMMO_PISTOL", 30)
player:removeWeaponAmmo(weaponIdentifier, "AMMO_PISTOL", 10)
player:setWeaponAmmo(weaponIdentifier, "AMMO_PISTOL", 100)
local ammo = player:getWeaponAmmo(weaponIdentifier, "AMMO_PISTOL")

-- สภาพอาวุธ
player:setWeaponCondition(weaponIdentifier, 80)
player:degradeWeapon(weaponIdentifier, 5)
player:repairWeapon(weaponIdentifier, 100)

-- Component
player:addWeaponComponent(weaponIdentifier, "slot", "value")
player:removeWeaponComponent(weaponIdentifier, "slot")
local components = player:getWeaponComponents(weaponIdentifier)

-- อื่นๆ
local has = player:hasWeapon("WEAPON_PISTOL")
local weapon = player:getWeapon(weaponIdentifier)
local weapons = player:getWeapons()
player:clearLoadout()
```

### Player Actions

```lua
player:kick("Reason here")
local playTime = player:getPlayTime()

-- บันทึกข้อมูล
player:save()

-- ลบตัวละคร
player:delete()
```

### Player State (Client)

```lua
-- ตรวจสอบ
local isLoaded = AXIS:IsPlayerLoaded()     -- true/false
local playerData = AXIS:GetPlayerData()    -- ข้อมูลทั้งหมด

-- ค้นหาใน inventory
local item = AXIS:SearchInventory("bread")
-- {name, label, count, weight, limit}

local items = AXIS:SearchInventory({"bread", "water"})
-- array of items

-- ตั้งค่า
AXIS:SetPlayerData("customKey", "customValue")

-- Coords (auto-tracking)
local coords = AXIS.PlayerData.coords  -- vector3
local ped = AXIS.PlayerData.ped         -- ped handle
```

---

## Membership System

ระบบสมาชิกแบบ Subscription พร้อม tier และวันหมดอายุ

### Server-side

```lua
local player = AXIS:GetPlayer(source)

-- ตั้ง membership
player:setMembership("Elite", 30)      -- Elite 30 วัน
player:setMembership("Ultimate", nil)   -- Ultimate ถาวร

-- อ่าน tier ปัจจุบัน (เช็ค expired อัตโนมัติ)
local tier = player:getMembership()     -- "Elite"

-- ลบ membership
player:removeMembership()               -- กลับเป็น Default
```

### Client-side

```lua
-- อ่าน membership ปัจจุบัน (เช็ค expired อัตโนมัติ)
local membership = AXIS:GetMyMembership()
-- {name = "Elite", label = "Elite", order = 1, color = "#50c878"}

if membership then
    print(membership.name)   -- "Elite"
    print(membership.order)  -- 1
    print(membership.color)  -- "#50c878"
end
```

### Shared Query

```lua
-- ข้อมูล tier ทั้งหมด
local tiers = AXIS:GetMemberships()     -- {"Default","Elite","Platinum","Ultimate"}

-- ข้อมูล tier เฉพาะ
local tier = AXIS:GetMembership("Elite")
-- {name = "Elite", label = "Elite", order = 1, color = "#50c878"}

-- Default tier
local default = AXIS:GetDefaultMembership()  -- "Default"

-- เปรียบเทียบ tier
local isHigher = AXIS:CompareMembership("Platinum", "Elite")  -- true (2 >= 1)
```

### Membership Config (`config/membership.lua`)

```lua
return {
    ["Default"] = {
        label = "Default",
        order = 0,
        color = "#ffffff",
    },
    ["Elite"] = {
        label = "Elite",
        order = 1,
        color = "#50c878",
    },
    -- ...
}
```

| Field | Type | Description |
|-------|------|-------------|
| `label` | string | ชื่อแสดงผล |
| `order` | number | ลำดับความสูง (0=ต่ำสุด) ใช้เปรียบเทียบ |
| `color` | string | สี hex สำหรับแสดงผล |

### เก็บใน metadata

```lua
-- Server ส่งผ่าน setMeta → sync ไป client อัตโนมัติ
-- ข้อมูลที่เก็บ:
metadata.membership = {
    tier = "Elite",
    expires = 1745892000,   -- unix timestamp (nil = ถาวร)
    purchased = 1743300000, -- วันที่ซื้อ
}
```

---

## Utility Modules

### AXIS.Math

```lua
AXIS.Math:Round(3.14159, 2)         -- 3.14
AXIS.Math:Clamp(150, 0, 100)         -- 100
AXIS.Math:Random(1, 10)              -- random int 1-10
AXIS.Math:RandomFloat(0.0, 1.0, 2)  -- 0.57
AXIS.Math:Percent(200, 25)           -- 50
AXIS.Math:IsPercent(50, 200, 25)     -- true
AXIS.Math:Lerp(0, 100, 0.5)         -- 50
AXIS.Math:Distance2D(x1,y1,x2,y2)   -- float
AXIS.Math:Distance3D(x1,y1,z1,x2,y2,z2) -- float
AXIS.Math:FormatNumber(1234567)      -- "1,234,567"
AXIS.Math:FormatCompact(1500000)     -- "1.5M"
```

### AXIS.Table

```lua
AXIS.Table:Contains(t, value)        -- boolean
AXIS.Table:ContainsKey(t, key)       -- boolean
AXIS.Table:IndexOf(t, value)         -- number?
AXIS.Table:Find(t, callback)         -- value, key
AXIS.Table:Filter(t, callback)       -- filtered array
AXIS.Table:Map(t, callback)          -- mapped array
AXIS.Table:Count(t)                  -- number
AXIS.Table:Merge(t1, t2)            -- merged table
AXIS.Table:Keys(t)                   -- array of keys
AXIS.Table:Values(t)                 -- array of values
AXIS.Table:Clone(t)                  -- shallow clone
AXIS.Table:DeepClone(t)              -- deep clone
AXIS.Table:RemoveValue(t, value)     -- remove all occurrences
AXIS.Table:Shuffle(t)                -- shuffled in-place
AXIS.Table:Slice(t, start, finish)   -- sub-table
AXIS.Table:Reverse(t)                -- reversed
AXIS.Table:First(t)                  -- t[1]
AXIS.Table:Last(t)                   -- t[#t]
AXIS.Table:IsEmpty(t)                -- boolean
AXIS.Table:Unique(t)                 -- deduplicated
AXIS.Table:Flatten(t, depth)         -- flattened
```

### AXIS.String

```lua
AXIS.String:Split("a,b,c", ",")       -- {"a","b","c"}
AXIS.String:Trim("  hello  ")          -- "hello"
AXIS.String:StartsWith("hello", "he")  -- true
AXIS.String:EndsWith("hello", "lo")    -- true
AXIS.String:Capitalize("hello")        -- "Hello"
AXIS.String:Upper("hello")             -- "HELLO"
AXIS.String:Lower("HELLO")             -- "hello"
AXIS.String:Replace("hello", "l", "r") -- "herro"
AXIS.String:Contains("hello", "ell")   -- true
AXIS.String:PadLeft("5", 3, "0")      -- "005"
AXIS.String:PadRight("hi", 5, " ")    -- "hi   "
AXIS.String:Truncate("hello world", 8) -- "hello..."
```

### AXIS.UUID

```lua
AXIS.UUID:Short()  -- "a1b2c3d4"
AXIS.UUID:Long()   -- "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
```

### AXIS.Time

```lua
AXIS.Time:Get()                          -- current unix timestamp (synced with server)
AXIS.Time:Diff(timestamp)               -- seconds since timestamp
AXIS.Time:FromSeconds(90061)            -- {days=1, hours=1, minutes=1, seconds=1}
AXIS.Time:FormatDuration(90061)         -- "1d 1h"
AXIS.Time:Format(timestamp, "%Y-%m-%d") -- "2025-01-01"
```

> Client ใช้เวลาจาก server (sync ผ่าน `CORE.BASE_TIME`) เพื่อความถูกต้อง

### อื่นๆ

```lua
-- Random string
AXIS:GetRandomString(8)  -- "xK3mP9qR"

-- Dump table
AXIS:DumpTable({a = 1, b = 2})  -- formatted string

-- Type validation
local ok, err = AXIS:ValidateType(42, "string")  -- false, "bad value..."
AXIS.AssertType(value, "number")                  -- assert

-- Load file
local config = AXIS.load("config.myfile")
local data = AXIS.loadJson("data/something")

-- Require
local module = AXIS.require("mymodule")
```

---

## Commands

### ระบบ RegisterCommand

```lua
AXIS:RegisterCommand(name, group, callback, allowConsole, suggestion)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | string \| table | ชื่อ command หรือ {"cmd1","cmd2"} |
| `group` | string \| table | สิทธิ์ เช่น "admin", {"admin","mod"} |
| `callback` | function | `function(player, args)` |
| `allowConsole` | boolean | ใช้จาก console ได้ไหม |
| `suggestion` | table | Chat suggestion + argument parsing |

### Argument Types

| Type | Description |
|------|-------------|
| `number` | แปลงเป็นตัวเลข |
| `string` | ตรวจสอบ string |
| `player` | คืน player object |
| `playerId` | คืน player ID (number) |
| `item` | ตรวจสอบว่ามี item |
| `weapon` | ตรวจสอบว่ามี weapon |
| `any` | รับอะไรก็ได้ |
| `merge` | รวม arguments ที่เหลือทั้งหมด |
| `coordinate` | แปลงเป็น coordinate |

> เมื่อใส่ `suggestion.arguments` args จะถูก parse เป็น named keys (`args.playerId`, `args.tier`)
> เมื่อ **ไม่ใส่** args จะเป็น array (`args[1]`, `args[2]`)

### Built-in Commands

| Command | Group | Description |
|---------|-------|-------------|
| `giveitem <id> <item> <amount>` | admin | ให้ไอเทม |
| `removeitem <id> <item> <amount>` | admin | เอาไอเทมออก |
| `giveweapon <id> <weapon>` | admin | ให้อาวุธ |
| `removeweapon <id> <weapon>` | admin | เอาอาวุธออก |
| `addammo <id> <weapon> <type> <amount>` | admin | เพิ่มกระสุน |
| `kick <id> [reason]` | admin | เตะผู้เล่น |
| `setgroup <id> <group>` | owner | ตั้ง group |
| `setjob <id> <job>` | admin | ตั้ง job |
| `setgrade <id> <grade>` | admin | ตั้ง grade |
| `setaccountmoney <id> <account> <amount>` | admin | ตั้งเงิน |
| `addaccountmoney <id> <account> <amount>` | admin | เพิ่มเงิน |
| `removeaccountmoney <id> <account> <amount>` | admin | ลดเงิน |
| `setcoords <id> <x> <y> <z>` | admin | เทเลพอร์ต |
| `clearinventory <id>` | admin | ล้าง inventory |
| `clearweapons <id>` | admin | ล้างอาวุธ |
| `setbucket <id> <bucket>` | admin | ตั้ง routing bucket |
| `goto <id>` | admin | เทเลพอร์ตไปหา |
| `bring <id>` | admin | เรียกมาหา |
| `die <id>` | admin | ฆ่า |
| `setmembership <id> <tier> [days]` | admin | ตั้ง membership |
| `rmembership <id>` | admin | ลบ membership |

---

## Configuration

### Config Files

| File | Description |
|------|-------------|
| `config/main.lua` | Spawn points, starter packs, accounts |
| `config/inventory.lua` | Items, max weight |
| `config/weapons.lua` | Weapons, ammo, categories, settings |
| `config/groups.lua` | Permission groups |
| `config/jobs.lua` | Jobs and grades |
| `config/membership.lua` | Membership tiers |

### Accessing Config

```lua
-- Items
local allItems = AXIS:GetItems()           -- array of all items
local item = AXIS:GetItems("bread")        -- single item data
local items = AXIS:GetItems({"bread","water"}) -- multiple

-- Jobs
local jobs = AXIS:GetJobs()                -- array of job names
local job = AXIS:GetJob("police")          -- job data
local defaultJob = AXIS:GetDefaultJob()    -- default job name

-- Groups
local groups = AXIS:GetGroups()            -- array of group names
local group = AXIS:GetGroup("admin")       -- group data
local defaultGroup = AXIS:GetDefaultGroup() -- default group name

-- Accounts
local account = AXIS:GetAccount("bank")    -- account data
local accounts = AXIS:GetAccounts()        -- array of account names

-- Weapons
local weapons = CORE.WEAPONS
local ammo = CORE.WEAPON_AMMO
local categories = CORE.WEAPON_CATEGORIES
local settings = CORE.WEAPON_SETTINGS
```

---

## Data Sync

### Server → Client Sync

Server sync ข้อมูลผ่าน `axis_core:client:sync`:

```lua
-- Server
TriggerClientEvent("axis_core:client:sync", source, key, value)
```

Client handler อัปเดต `AXIS.PlayerData[key]` อัตโนมัติ:

| Key | Behavior |
|-----|----------|
| `"inventory"` | แทนที่ inventory + อัปเดต weight |
| `"inventoryItem"` | อัปเดต item เดียว + weight |
| `"account"` | อัปเดต account เดียว |
| `"job"` | อัปเดต job + grade |
| อื่นๆ | `AXIS.PlayerData[key] = value` |

### Sync Multiple

```lua
TriggerClientEvent("axis_core:client:syncMultiple", source, {
    job = {job = "police", job_grade = 3},
    metadata = {health = 200, membership = {tier = "Elite"}},
})
```

### OnPlayerData Callback

```lua
-- ใน resource อื่น (หลัง imports.lua)
OnPlayerData = function(key, val, last)
    if key == "metadata" then
        print("Metadata updated!")
    end
end
```

---

## Internal Events

Events เหล่านี้ใช้ภายใน ไม่ควรเรียกโดยตรง:

| Event | Direction | Description |
|-------|-----------|-------------|
| `axis_core:server:event` | Client → Server | ส่ง event |
| `axis_core:server:callbackEvent` | Server → Server | ส่ง callback กลับ |
| `axis_core:client:event` | Server → Client | ส่ง event |
| `axis_core:client:callbackEvent` | Client → Client | ส่ง callback กลับ |
| `axis_core:client:sync` | Server → Client | sync ข้อมูล |
| `axis_core:client:syncMultiple` | Server → Client | sync หลาย key |
| `axis_core:server:playerLoaded` | Server | player load สำเร็จ |
| `axis_core:server:requestTime` | Client → Server | ขอเวลาจาก server |
| `axis_core:server:updateHealth` | Client → Server | sync ค่า health |
| `axis_core:server:syncAmmo` | Client → Server | sync กระสุน |
| `axis_core:server:sync` | Client → Server | sync ข้อมูลทั่วไป |

---

## Usable Items

```lua
-- ลงทะเบียน item ที่ใช้ได้
AXIS:RegisterUsableItem("bandage", function(source, item)
    local player = AXIS:GetPlayer(source)
    player:addAccountMoney("bank", 100)
end)

-- เช็ค
local isUsable = AXIS:IsUsableItem("bandage")
local allUsable = AXIS:GetUsableItems()
```

---

## Flow Diagram

### Server → Client → Server

```
Server                              Client
  |                                   |
  | AXIS:TriggerClientEvent()         |
  |---------------------------------->|
  |                                   | AXIS:RegisterNetEvent() handler
  |                                   | return value
  |<----------------------------------|
  | callback(source, ...)             |
```

### Client → Server → Client

```
Client                              Server
  |                                   |
  | AXIS:TriggerServerEvent()         |
  |---------------------------------->|
  |                                   | AXIS:RegisterNetEvent() handler
  |                                   | return value
  |<----------------------------------|
  | callback(...)                     |
```

### Data Sync Flow

```
Server                              Client
  |                                   |
  | setMeta("membership", {...})       |
  |       |                           |
  |       v                           |
  | TriggerClientEvent("sync")        |
  |---------------------------------->|
  |                                   | AXIS.PlayerData.metadata = value
  |                                   | OnPlayerData("metadata", ...)
```

---

## Error Handling

### Timeout

```lua
AXIS:TriggerServerEvent({
    name = "slowOperation",
    timeout = 3000
})
-- ถ้า server ไม่ตอบภายใน 3 วินาที:
-- "[axis_core] Event 'slowOperation' timed out without response"
```

### Event Not Registered

```lua
-- ถ้าไม่มี handler สำหรับ "unknownEvent"
-- Error: "No callback found for event unknownEvent"
```

---

## Best Practices

1. **ใช้ชื่อ event ให้สื่อความหมาย** — `"getPlayerMoney"` ไม่ใช่ `"event1"`
2. **ตั้ง timeout ตาม operation** — เร็ว = 1000ms, DB query = 5000ms
3. **จัดการ nil ใน callback** — เช็ค `if not data then return end`
4. **Return ค่าที่มีความหมาย** — `return success, message`
5. **ใช้ `setMeta` / `getMeta`** แทนการยุ่งกับ `metadata` ตรงๆ
6. **ใช้ `AXIS.Time:Get()`** บน client แทน `os.time()` (เพราะ client ไม่มี `os.time`)
