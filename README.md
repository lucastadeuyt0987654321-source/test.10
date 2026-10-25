--[[
============================================================
  Private Script - v18.12.15
  + Interface principal minimiza para imagem
  + Anti-AFK minimiza para quadrado flutuante
  + Dano total do Auto Boss na tela
  + UI redesenhada (visual moderno)
  + FIX: travamentos removidos (RenderStepped throttle)
  + FIX: findGlobalPart cacheado
  + FIX: Player.Idled conectado uma vez só
  + FIX HEAD: Auto Boss colado na cabeça
  + BTN BG: imagem de fundo (ID: 92343722959174)
  + MOVE: Auto Boss e Farm abaixo do Auto Punch Fast
  + UI BG: imagem de fundo da interface (ID: 107833219559616)
  + FECHAR NO X DO LAUNCHER = DESLIGA SCRIPT COMPLETO
  + TP -> Teleport
  + SEPARADORES VERTICAIS entre as ABAS DO TOPO
  + FIX: Boss.coletarBau otimizado e seguro
  + NOVO: Aba "Strong Bloxian Store" (Auto Buy Medal Shop)
  + FIX: Auto Buy teleporta apenas se houver Pet com estoque e saldo suficiente
  + NOVO: Sistema de cor Auto RGB que combina com o fundo
  + REMOVIDO: Botão "Auto Race"
  + FIX STORE: Corrigido conforme MedalShopController decompilado
  + FIX CRASH: Loja em pcall + RenderStepped throttle 20Hz
  + STORE: Teleporta para workspace.MedalShop + só compra com saldo
  + STORE: Moeda correta = MedalCoin
============================================================a
]]

--==========================================================
-- 0. CLEANUP
--==========================================================
shared = shared or {}
if shared.__PS_TOKEN then shared.__PS_TOKEN.dead = true end
local TOKEN = { dead = false }
shared.__PS_TOKEN = TOKEN

local Players              = game:GetService("Players")
local RunService           = game:GetService("RunService")
local TweenService         = game:GetService("TweenService")
local UserInputService     = game:GetService("UserInputService")
local VirtualInputManager  = game:GetService("VirtualInputManager")
local Lighting             = game:GetService("Lighting")
local ReplicatedStorage    = game:GetService("ReplicatedStorage")
local VirtualUser          = game:GetService("VirtualUser")
local LogService           = game:GetService("LogService")

local Player = Players.LocalPlayer
local PARENT_GUI = RunService:IsStudio() and Player.PlayerGui
                   or (gethui and gethui() or game:GetService("CoreGui"))

local CLEANUP_NAMES = {
    "PrivateScriptMainGui","PS_FloatingBtn","LauncherGui","AutoFastGui",
    "PS_StatsGui","PS_Diag","PS_Diagnostic","PS_ClickVisual",
    "PS_MainGui","PS_Launcher","PS_FastGui","PrivateScriptGui","PS_Hub",
}
local function cleanupGuis(container)
    if not container then return end
    for _, name in ipairs(CLEANUP_NAMES) do
        pcall(function()
            local g = container:FindFirstChild(name)
            if g then g:Destroy() end
        end)
    end
    pcall(function()
        for _, g in ipairs(container:GetChildren()) do
            if g:IsA("ScreenGui") then
                local n = g.Name
                if n:sub(1,3) == "PS_" or n:sub(1,13) == "PrivateScript" then
                    g:Destroy()
                end
            end
        end
    end)
end
cleanupGuis(PARENT_GUI)
cleanupGuis(Player:FindFirstChild("PlayerGui"))
pcall(function() if gethui then cleanupGuis(gethui()) end end)
pcall(function() cleanupGuis(game:GetService("CoreGui")) end)

local statsGuiRef = nil
local idledConnected = false

--==========================================================
-- 0B. LIMPAR CONSOLE
--==========================================================
pcall(function() LogService:ClearOutput() end)

task.spawn(function()
    while not TOKEN.dead do
        task.wait(3)
        pcall(function() LogService:ClearOutput() end)
    end
end)

--==========================================================
-- 0C. SILENCIADOR
--==========================================================
pcall(function()
    local IGNORAR = {
        "Invalid entry", "parameter name", "Script timeout",
        "exhausted allowed execution time",
        "ReplicatedStorage.Storage.clientUtils",
        "ReplicatedStorage.Storage.client.controllers",
        "ReplicatedStorage.packages.Loader",
        "ReplicatedStorage.Storage.client.bootstrapper",
    }
    local function deveIgnorar(msg)
        if not msg then return false end
        for _, kw in ipairs(IGNORAR) do
            if string.find(msg, kw, 1, true) then return true end
        end
        return false
    end
    LogService.MessageOut:Connect(function(msg, msgType)
        if msgType == Enum.MessageType.MessageError
        or msgType == Enum.MessageType.MessageWarning
        or msgType == Enum.MessageType.MessageInfo then
            if deveIgnorar(msg) then
                pcall(function() LogService:ClearOutput() end)
            end
        end
    end)
end)

--==========================================================
-- 1. CONFIG
--==========================================================
local CFG = {
    FONT             = Enum.Font.GothamBold,
    FAST_SPEED       = 3,
    PUNCH_RATE       = 0.05,
    PUNCH_FAST_SPEED = 20,
    SEARCH_RATE      = 0.5,
    HEAD_OFFSET      = 6,
    CHEST_HEAD_OFFSET= 3,
    HW_HEAD_OFFSET   = 6,
    HITBOX_SIZE      = 25,
    HITBOX_DOWN_OFFSET = 6,
    ARENA_FALLBACK   = Vector3.new(-36, 65, -1262),
    REBIRTH_RATE     = 1,
    CHEST_MAX_TIME   = 20,
    CHEST_CACHE_T    = 0.5,
    HP_UPDATE_RATE   = 0.1,
    FLOAT_ICON_ID    = "rbxassetid://132131598840017",
    LAUNCHER_ICON_ID = "rbxassetid://125445528906346",
    AUTO_FAST_DELAY  = 0.01,
    REP_BATCH        = 10,
    TITLE            = "PRIVATE SCRIPT",
    TITLE_WELCOME    = "Bem-vindo, " .. Player.Name,
    AUTO_SIZE_RATE   = 2,
    GIFT_INTERVAL    = 5,
    GIFT_TOTAL       = 8,
    FORTUNE_INTERVAL = 1,
    GAMEPASS_INTERVAL = 2,
    STATS_UPDATE_RATE = 0.5,
    BANNER_IMAGE     = "rbxassetid://127400351441808",
    BAT_IMAGE        = "rbxassetid://98802974958599",
    WITCH_IMAGE      = "rbxassetid://79857720013778",
    HEADER_HEIGHT    = 0,
    BG_IMAGE         = "rbxassetid://127400351441808",
    UI_BG_IMAGE      = "rbxassetid://107833219559616",
    DISCORD_ICON     = "rbxassetid://111762867350633",
    LAUNCHER_BG      = "rbxassetid://131059064247989",
    BTN_BG_IMAGE     = "rbxassetid://92343722959174",
}

local MUSCLE_KING_DEFAULT_POS = Vector3.new(-8626, 17, -5730)
local HW_ISLAND_POS = Vector3.new(5556.76513671875, 115.29641723632812, 5844.65771484375)
local BOSS_WAIT_POS = Vector3.new(-36, 65, -1262)

local MACHINE_KEYWORDS = {
    "squat","bench","pullup","pull-up","pullup","deadlift","dead lift",
    "boulder","throw","barbell","press","rack",
}
local TOOL_KEYWORDS = {
    "weight","peso","situacao","situações","situp","situps","pushup","pushups",
    "flexao","flexões","handstand","handstands","punch","soco","fist","stomp",
    "pisao","pisão","suportes","supports","support","golpe no chao","golpe no chão",
    "ground","groundpunch","kick","chute","chutar","train","training","treino",
    "treinar","barbell","bar","dumbbell","dumbell","boulder","rock","pedra",
    "lift","lifting","gym",
}
local KEYWORDS = {
    PUNCH         = {"punch","soco","fist","combat","hit","punching"},
    MOVEMENT_ANIM = {"idle","walk","run","jump","fall","climb","swim","freefall",
                     "land","toolnone","tool grip","sit","pose","dance"},
    CHEST         = {"goldenchest","enchantedchest","junglechest","legendschest",
                     "industrialchest","groupchest","magmachest","mythicalchest",
                     "overchargedchest","bosschest","rewardchest","chest","bau"},
    TP_BAD        = {"tobeach","tospawn","press","machine","frame","cylinder","npc",
                     "lefthand","righthand","leftarm","rightarm","head","mainglass",
                     "maingem","crystal","orb","chest","trigger","hitbox","zone"},
    REBIRTH       = {"rebirth","renasc","prestige"},
    CHEST_OPENED  = {"open","loot","collect","aberto","coletado","used"},
}

--==========================================================
-- 2. STATE
--==========================================================
local State = {
    autoTP   = { active = false, lastTP = 0, island = "?", keepLock = false },
    lockPos  = { active = false, pos = nil },
    lastWindow = "main",
    antiAfk  = { active = false, startTime = 0, totalTime = 0 },
    autoPunch = { active = false, lastPunch = 0 },
    autoBoss = {
        active = false, mode = "idle", target = nil, targetHum = nil,
        arenaPos = BOSS_WAIT_POS, hitboxPart = nil,
        chestTPDone = false, chestStartTime = 0, chestWaitStart = 0,
        chestCount = 0, chestJiggleIdx = 0,
        hasTeleported = false,
        savedArenaPos = nil,
        customArenaPos = BOSS_WAIT_POS,
        currentHp = 0, maxHp = 0, currentBossName = "",
        offsetY = 0,
        lastBossTP = 0,
        lastChestTP = 0,
        _lastPartPrint = 0,
        chestCollected = false,
        totalDamage = 0,
        lastBossHp = 0,
        damageDisplay = nil,
    },
    tools = {
        Weight     = { auto = false, fast = false },
        Situps     = { auto = false, fast = false },
        Pushups    = { auto = false, fast = false },
        Handstands = { auto = false, fast = false },
    },
    autoBossFarm = { active = false, phase = "idle" },
    autoPrestige = { active = false, lastFire = 0 },
    autoMedalShop = { active = false, currentIndex = 3 },
    rgbMode = false,
}

local HW_BOSSES = {
    { id="scarecrow", name="Scarecrow Colossus", folder="Scarecrow Colossus",
      diff="Easy", color=Color3.fromRGB(120,200,120),
      active=false, target=nil, mode="idle",
      offsetY=0, hp=0, maxHp=0, hitbox=nil,
      spawnPos = Vector3.new(5145.67, 63.42, 4688.78)
    },
    { id="pumpkin", name="Pumpkin Titan", folder="Pumpkin Titan",
      diff="Medium", color=Color3.fromRGB(255,150,50),
      active=false, target=nil, mode="idle",
      offsetY=0, hp=0, maxHp=0, hitbox=nil,
      spawnPos = Vector3.new(5151.50, 168.03, 7844.78)
    },
    { id="franken", name="Frankenbrute", folder="Frankenbrute",
      diff="Hard", color=Color3.fromRGB(180,100,255),
      active=false, target=nil, mode="idle",
      offsetY=0, hp=0, maxHp=0, hitbox=nil,
      spawnPos = Vector3.new(6732.27, 63.04, 6275.88)
    },
    { id="reaper", name="Grim Reaper", folder="Grim Reaper",
      diff="Extreme", color=Color3.fromRGB(255,50,50),
      active=false, target=nil, mode="idle",
      offsetY=0, hp=0, maxHp=0, hitbox=nil,
      spawnPos = Vector3.new(3599.71, 167.60, 6278.40)
    },
}

local AutoFast, AutoRebirth, AutoTP
local AutoClaimGift = { active=false, lastClaim=0, total=0, status="Off", proximoGift=1 }
local AutoFortune   = { active=false, lastSpin=0, total=0, status="Off" }
local AutoGamepass  = { active=false, lastFire=0, total=0, status="Off" }
local AutoSize2     = { active=false, lastApply=0, lastError=nil }
local AutoSize10    = { active=false, lastApply=0, lastError=nil }
local BossHpLabel, BossHpBar, BossHpBarFill, BossToggleBtn
local BossFarmToggleBtn, BossFarmStroke
local HW_HpLabel, HW_HpBar, HW_HpBarFill

local AutoClaimQuest = { active = false, lastFire = 0 }
local AutoBypass     = { active = false, lastFire = 0, currentTier = 1 }
local AutoPrestige   = { active = false, lastFire = 0 }

--==========================================================
-- 3. UTILS
--==========================================================
local Utils = {}
function Utils.alive() return not TOKEN.dead end
function Utils.isMobile() return UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled end

function Utils.scaledSize(baseW, baseH)
    local cam = workspace.CurrentCamera
    local vp = cam and cam.ViewportSize or Vector2.new(1920, 1080)
    local maxW = vp.X * 0.92
    local maxH = vp.Y * 0.75
    return UDim2.new(0, math.floor(math.min(baseW, maxW)),
                     0, math.floor(math.min(baseH, maxH)))
end

function Utils.create(class, props, parent)
    local inst = Instance.new(class)
    for k, v in pairs(props) do
        pcall(function() inst[k] = v end)
    end
    if parent then pcall(function() inst.Parent = parent end) end
    return inst
end

function Utils.lower(s) return string.lower(s or "") end
function Utils.containsAny(str, list)
    local l = Utils.lower(str)
    for _, kw in ipairs(list) do
        if l:find(kw, 1, true) then return true end
    end
    return false
end

function Utils.formatNumber(n)
    if not n then return "0" end
    local num = tonumber(n)
    if not num then return tostring(n) end
    local sinal = ""
    if num < 0 then sinal = "-"; num = -num end
    if num < 1000 then
        if num % 1 == 0 then return sinal .. tostring(math.floor(num)) end
        return sinal .. tostring(num)
    end
    local sufixos = {
        "", "K", "M", "B", "T",
        "Qa", "Qi", "Sx", "Sp", "Oc", "No", "Dc",
        "Ud", "Dd", "Td", "Qad", "Qid", "Sxd", "Spd", "Od", "Nd", "Vg",
        "Uvg", "Dvg", "Tvg", "Qavg", "Qivg", "Sxvg", "Spvg", "Ovg", "Nvg", "Tg",
        "Utg", "Dtg", "Ttg", "Qatg", "Qitg", "Sxtg", "Sptg", "Otg", "Ntg", "Qd",
        "Uqd", "Dqd", "Tqd", "Qaqd", "Qiqd", "Sxqd", "Spqd", "Oqd", "Nqd", "Qq",
        "Uqq", "Dqq", "Tqq", "Qaqq", "Qiqq", "Sxqq", "Spqq", "Oqq", "Nqq", "Sq",
        "Usq", "Dsq", "Tsq", "Qasq", "Qisq", "Sxsq", "Spsq", "Osq", "Nsq", "St",
        "Ust", "Dst", "Tst", "Qast", "Qist", "Sxst", "Spst", "Ost", "Nst", "Og",
        "Uog", "Dog", "Tog", "Qaog", "Qiog", "Sxog", "Spog", "Oog", "Nog", "Ng",
        "Ung", "Dng", "Tng", "Qang", "Qing", "Sxng", "Spng", "Ong", "Nng", "Cg",
    }
    local i = 1
    while num >= 1000 and i < #sufixos do
        num = num / 1000
        i = i + 1
    end
    local extra = ""
    if num >= 1000 then
        local expoente = math.floor(math.log(num, 10)) + (i - 1) * 3
        num = num / (10 ^ math.floor(math.log(num, 10)))
        extra = "x10^" .. tostring(expoente)
    end
    local texto = string.format("%.2f", num)
    texto = texto:gsub("%.00$", "")
    texto = texto:gsub("(%.%d)0$", "%1")
    return sinal .. texto .. sufixos[i] .. extra
end

function Utils.isAllowedMachine(model)
    if not model then return false, nil, nil end
    local name = Utils.lower(model.Name)
    local parentName = ""
    if model.Parent and model.Parent ~= workspace then
        parentName = Utils.lower(model.Parent.Name)
    end
    local idx = nil
    local machinesFolder = workspace:FindFirstChild("machinesFolder")
    if machinesFolder then
        local children = machinesFolder:GetChildren()
        for i, child in ipairs(children) do
            if child == model or child:IsAncestorOf(model) or model:IsAncestorOf(child) then
                idx = i
                if name == "" then name = Utils.lower(child.Name) end
                break
            end
        end
    end
    local searchName = name .. " " .. parentName
    if Utils.containsAny(searchName, MACHINE_KEYWORDS) then
        return true, "Name:" .. name, idx
    end
    for _, child in ipairs(model:GetChildren()) do
        local cn = Utils.lower(child.Name)
        if cn == "barbell" or cn == "boulder" then
            return true, "Child:" .. child.Name, idx
        end
    end
    return false, nil, idx
end

function Utils.isAllowedTool(tool)
    if not tool or not tool:IsA("Tool") then return false, nil end
    local name = Utils.lower(tool.Name)
    if Utils.containsAny(name, TOOL_KEYWORDS) then return true, name end
    return false, nil
end

function Utils.signalClick(obj)
    if not obj then return end
    pcall(function() obj:Activate() end)
    if type(firesignal) == "function" then
        pcall(function() firesignal(obj.Activated) end)
        pcall(function() firesignal(obj.MouseButton1Click) end)
        pcall(function() firesignal(obj.MouseButton1Down) end)
        pcall(function() firesignal(obj.MouseButton1Up) end)
    end
end

function Utils.getHRP()
    local char = Player.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart")
end

function Utils.getPlayerImage()
    local img = "rbxassetid://132131598840017"
    pcall(function()
        img = Players:GetUserThumbnailAsync(Player.UserId,
            Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size150x150)
    end)
    return img
end

function Utils.makeDraggable(bar, container)
    local dragging, input, start, startPos
    bar.InputBegan:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseButton1
        or inp.UserInputType == Enum.UserInputType.Touch then
            dragging = true; start = inp.Position; startPos = container.Position
            inp.Changed:Connect(function()
                if inp.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    bar.InputChanged:Connect(function(inp)
        if inp.UserInputType == Enum.UserInputType.MouseMovement
        or inp.UserInputType == Enum.UserInputType.Touch then input = inp end
    end)
    UserInputService.InputChanged:Connect(function(inp)
        if inp == input and dragging then
            local d = inp.Position - start
            container.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X,
                startPos.Y.Scale, startPos.Y.Offset + d.Y)
        end
    end)
end

function Utils.getHalloweenPassRemote()
    local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
    if rEvents then
        local remote = rEvents:FindFirstChild("halloweenPassRemote")
        if remote and (remote:IsA("RemoteFunction") or remote:IsA("RemoteEvent")) then
            return remote
        end
    end
    for _, container in ipairs({ReplicatedStorage, Player}) do
        for _, obj in ipairs(container:GetDescendants()) do
            if obj.Name == "halloweenPassRemote"
               and (obj:IsA("RemoteFunction") or obj:IsA("RemoteEvent")) then
                return obj
            end
        end
    end
    return nil
end

function Utils.addBtnBg(parent, transparency, color)
    if not parent then return end
    pcall(function() parent.ClipsDescendants = true end)
    local bg = Utils.create("ImageLabel", {
        BackgroundTransparency = 1,
        Image = CFG.BTN_BG_IMAGE,
        ImageTransparency = transparency or 0.15,
        ImageColor3 = color or Color3.fromRGB(255, 255, 255),
        ScaleType = Enum.ScaleType.Crop,
        Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, 0, 1, 0),
        ZIndex = 2,
    }, parent)
    pcall(function()
        local corner = parent:FindFirstChildOfClass("UICorner")
        if corner then
            local c = Instance.new("UICorner")
            c.CornerRadius = corner.CornerRadius
            c.Parent = bg
        end
    end)
    return bg
end

function Utils.addUiBackground(parent, transparency)
    if not parent then return end
    pcall(function() parent.ClipsDescendants = true end)
    local bg = Utils.create("ImageLabel", {
        BackgroundTransparency = 1,
        Image = CFG.UI_BG_IMAGE,
        ImageTransparency = transparency or 0.75,
        ScaleType = Enum.ScaleType.Crop,
        Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, 0, 1, 0),
        ZIndex = 0,
    }, parent)
    pcall(function()
        local corner = parent:FindFirstChildOfClass("UICorner")
        if corner then
            local c = Instance.new("UICorner")
            c.CornerRadius = corner.CornerRadius
            c.Parent = bg
        end
    end)
    return bg
end

--==========================================================
-- TP HALLOWEEN ISLAND
--==========================================================
function TP_HalloweenIsland(bossId)
    local hrp = Utils.getHRP()
    if not hrp then
        warn("[TP Halloween] Character nao encontrado")
        return
    end
    local targetBoss = nil
    for _, b in ipairs(HW_BOSSES) do
        if b.id == bossId then
            targetBoss = b
            break
        end
    end
    if not targetBoss then
        warn("[TP Halloween] Boss nao encontrado para TP")
        return
    end
    local targetPos = targetBoss.spawnPos
    pcall(function()
        hrp.CFrame = CFrame.new(targetPos)
        hrp.Velocity = Vector3.zero
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.RotVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
    end)
    task.wait(0.1)
    pcall(function() hrp.CFrame = CFrame.new(targetPos) end)
    print("[TP Halloween] Teleportado para " .. targetBoss.name .. " em " .. tostring(targetPos))
end

--==========================================================
-- 4. ANIM
--==========================================================
local Anim = {}
function Anim.isPunchTool(tool)
    return tool and Utils.containsAny(tool.Name, KEYWORDS.PUNCH) or false
end
function Anim.isMovementAnim(name, id)
    return Utils.containsAny((name or "") .. " " .. (id or ""), KEYWORDS.MOVEMENT_ANIM)
end
function Anim.currentToolType()
    local char = Player.Character
    if not char then return nil end
    local tool = char:FindFirstChildOfClass("Tool")
    if not tool then return nil end
    local n = Utils.lower(tool.Name)
    if n:find("weight") or n:find("peso") or n:find("dumbbell")
    or n:find("barbell") or n:find("lift")   then return "Weight"     end
    if n:find("situp") or n:find("situacao") then return "Situps"     end
    if n:find("pushup")    then return "Pushups"    end
    if n:find("handstand") then return "Handstands" end
    return nil
end
function Anim.anyEnabled()
    for _, s in pairs(State.tools) do
        if s.auto or s.fast then return true end
    end
    return false
end
function Anim.applyEffect(track)
    if not track then return end
    local a = track.Animation
    local nm = a and a.Name or ""
    local id = a and tostring(a.AnimationId) or ""
    if Anim.isMovementAnim(nm, id) then return end
    local t = Anim.currentToolType()
    if not t then return end
    local s = State.tools[t]
    if s.fast then
        pcall(function() track:AdjustSpeed(CFG.FAST_SPEED) end)
    elseif s.auto then
        pcall(function() track:Stop(0.1) end)
    end
end

local animKillerConn
function Anim.setupKiller()
    local char = Player.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local anim = hum and hum:FindFirstChildOfClass("Animator")
    if not anim then return end
    if animKillerConn then pcall(function() animKillerConn:Disconnect() end) end
    animKillerConn = anim.AnimationPlayed:Connect(function(track)
        if Anim.anyEnabled()
           or (AutoFast and (AutoFast.active or AutoFast.passive))
           or (State.autoBossFarm and State.autoBossFarm.active) then
            Anim.applyEffect(track)
        end
    end)
end

Player.CharacterAdded:Connect(function() task.wait(0.5); Anim.setupKiller() end)
if Player.Character then Anim.setupKiller() end

--==========================================================
-- 6. AUTO FAST
--==========================================================
AutoFast = {
    active = false, mode = "none", target = nil, detectMethod = "",
    cps = 0, totalCalls = 0, muscleEvent = nil, lastRepFind = 0,
    repRunning = false, repThread = nil, repGeneration = 0,
    _lastRemoteWarn = 0, _lastErrWarn = 0,
}

function AutoFast.findMuscleEvent()
    local now = tick()
    if AutoFast.muscleEvent and AutoFast.muscleEvent.Parent
       and AutoFast.muscleEvent:IsA("RemoteEvent")
       and (now - AutoFast.lastRepFind) < 5 then
        return AutoFast.muscleEvent
    end
    AutoFast.lastRepFind = now
    local event = Player:FindFirstChild("muscleEvent")
    if event and event:IsA("RemoteEvent") then
        AutoFast.muscleEvent = event
        return event
    end
    for _, obj in ipairs(Player:GetDescendants()) do
        if obj:IsA("RemoteEvent") and string.lower(obj.Name) == "muscleevent" then
            AutoFast.muscleEvent = obj
            return obj
        end
    end
    AutoFast.muscleEvent = nil
    return nil
end

function AutoFast.findToolRemote() return AutoFast.findMuscleEvent() end

function AutoFast.fireRepBatch()
    local event = AutoFast.findMuscleEvent()
    if not event then
        if tick() - AutoFast._lastRemoteWarn > 3 then
            AutoFast._lastRemoteWarn = tick()
            warn("[AutoFast] Player.muscleEvent nao encontrado.")
        end
        return 0
    end
    local batch = math.max(1, math.floor(tonumber(CFG.REP_BATCH) or 1))
    local fired = 0
    for _ = 1, batch do
        local ok, err = pcall(function() event:FireServer("rep") end)
        if not ok then
            if tick() - AutoFast._lastErrWarn > 3 then
                AutoFast._lastErrWarn = tick()
                warn("[AutoFast] Erro no rep:", err)
            end
            break
        end
        fired = fired + 1
    end
    return fired
end

function AutoFast.startRepLoop()
    if AutoFast.repRunning then return end
    AutoFast.repRunning = true
    AutoFast.repGeneration = AutoFast.repGeneration + 1
    local generation = AutoFast.repGeneration
    AutoFast.repThread = task.spawn(function()
        while AutoFast.repRunning and AutoFast.repGeneration == generation and Utils.alive() do
            if AutoFast.active then
                local fired = AutoFast.fireRepBatch()
                AutoFast.cps = AutoFast.cps + fired
                AutoFast.totalCalls = AutoFast.totalCalls + fired
                AutoFast.mode = "rep"
                AutoFast.target = tostring(CFG.REP_BATCH) .. " reps / " ..
                    tostring(CFG.AUTO_FAST_DELAY) .. "s"
                AutoFast.detectMethod = "muscleEvent:FireServer(\"rep\")"
            end
            task.wait(math.max(tonumber(CFG.AUTO_FAST_DELAY) or 0.01, 0))
        end
        if AutoFast.repGeneration == generation then
            AutoFast.repRunning = false
            AutoFast.repThread = nil
        end
    end)
end

function AutoFast.stopRepLoop()
    AutoFast.repRunning = false
    AutoFast.repGeneration = AutoFast.repGeneration + 1
    AutoFast.repThread = nil
    AutoFast.mode = "none"
    AutoFast.target = nil
    AutoFast.detectMethod = ""
end

function AutoFast.tick()
    if not Utils.alive() then return end
    if AutoFast.active then AutoFast.startRepLoop()
    else AutoFast.stopRepLoop() end
end

task.spawn(function()
    while Utils.alive() do
        pcall(AutoFast.tick)
        task.wait(0.1)
    end
end)

--==========================================================
-- 7. AUTO TP
--==========================================================
AutoTP = { active=false, lastTP=0, customPos=MUSCLE_KING_DEFAULT_POS, useCustom=true }

local findCache = { results = {}, time = 0 }

local function findIslandModel(kw)
    local lkw = Utils.lower(kw)
    for _, child in ipairs(workspace:GetChildren()) do
        if child ~= Player.Character then
            if Utils.lower(child.Name):find(lkw, 1, true) then
                for _, d in ipairs(child:GetDescendants()) do
                    if d:IsA("BasePart") and Utils.lower(d.Name):find("islandmodel", 1, true) then
                        return d
                    end
                end
            end
        end
    end
    return nil
end

local function findGlobalPart(kw)
    local now = tick()
    local cacheKey = kw
    if now - findCache.time < 5 and findCache.results[cacheKey] then
        local cached = findCache.results[cacheKey]
        if cached.Parent then return cached end
    end

    local myChar = Player.Character
    local lkw = Utils.lower(kw)
    local best, bestScore = nil, 0
    local count = 0
    for _, obj in ipairs(workspace:GetDescendants()) do
        count = count + 1
        if count % 200 == 0 then task.wait() end
        if obj:IsA("BasePart") and not (myChar and obj:IsDescendantOf(myChar)) then
            local fp = obj:GetFullName()
            local ln = Utils.lower(obj.Name)
            local lp = Utils.lower(fp)
            if not Utils.containsAny(ln, KEYWORDS.TP_BAD)
               and not Utils.containsAny(lp, KEYWORDS.TP_BAD) then
                if ln:find(lkw,1,true) or lp:find(lkw,1,true) then
                    local sc = #lkw
                    if ln:find("islandmodel",1,true) then sc = sc + 200 end
                    if ln:find("floor",1,true)       then sc = sc + 80  end
                    if lp:find("island",1,true)      then sc = sc + 50  end
                    if sc > bestScore then bestScore = sc; best = obj end
                end
            end
        end
    end

    findCache.time = now
    findCache.results[cacheKey] = best
    return best
end

function AutoTP.findUp(kw)
    return findIslandModel(kw) or findGlobalPart(kw)
end

function AutoTP.teleportToMuscle()
    local hrp = Utils.getHRP()
    if not hrp then return end
    local targetPos
    if AutoTP.useCustom then
        targetPos = AutoTP.customPos
        State.autoTP.island = string.format("Custom (%.0f, %.0f, %.0f)",
            targetPos.X, targetPos.Y, targetPos.Z)
    else
        local p = AutoTP.findUp("Muscle")
        if p then
            targetPos = p.Position + Vector3.new(0, 5, 0)
            State.autoTP.island = (p.Parent and p.Parent.Name) or "Muscle King"
        else
            State.autoTP.island = "Muscle King nao encontrado"
            return
        end
    end
    pcall(function()
        hrp.CFrame = CFrame.new(targetPos)
        hrp.Velocity = Vector3.zero        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.RotVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
    end)
end

task.spawn(function()
    while Utils.alive() do
        pcall(function()
            if AutoTP.active then
                local now = tick()
                if now - AutoTP.lastTP >= 1 then
                    AutoTP.lastTP = now
                    AutoTP.teleportToMuscle()
                end
            end
        end)
        task.wait(0.2)
    end
end)

task.spawn(function()
    while Utils.alive() do
        pcall(function()
            if State.autoTP.keepLock and AutoTP.active then
                local hrp = Utils.getHRP()
                if hrp then
                    local targetPos = AutoTP.customPos
                    local diff = (hrp.Position - targetPos).Magnitude
                    if diff > 3 then
                        hrp.CFrame = CFrame.new(targetPos)
                        hrp.Velocity = Vector3.zero
                        hrp.AssemblyLinearVelocity = Vector3.zero
                        hrp.RotVelocity = Vector3.zero
                        hrp.AssemblyAngularVelocity = Vector3.zero
                    end
                end
            end
        end)
        task.wait(0.15)
    end
end)

--==========================================================
-- 8. LOCK POSITION
--==========================================================
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            if State.lockPos.active and State.lockPos.pos then
                local hrp = Utils.getHRP()
                if hrp then
                    local diff = (hrp.Position - State.lockPos.pos).Magnitude
                    if diff > 1 then
                        local look = hrp.CFrame.LookVector
                        hrp.CFrame = CFrame.lookAt(State.lockPos.pos, State.lockPos.pos + look)
                        hrp.Velocity = Vector3.zero
                        hrp.AssemblyLinearVelocity = Vector3.zero
                    end
                end
            end
        end)
        task.wait(0.1)
    end
end)

--==========================================================
-- 9. AUTO REBIRTH
--==========================================================
AutoRebirth = { active=false, lastFire=0, status="Aguardando", remotes={} }

function AutoRebirth.findRebirthRemote()
    local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
    if rEvents then
        local remote = rEvents:FindFirstChild("rebirthRemote")
        if remote and (remote:IsA("RemoteFunction") or remote:IsA("RemoteEvent")) then
            return remote, "rEvents.rebirthRemote"
        end
    end
    if #AutoRebirth.remotes > 0 then
        for _, r in ipairs(AutoRebirth.remotes) do
            if r and r.Parent then return r, r.Name end
        end
    end
    AutoRebirth.remotes = {}
    local search = {
        Player, ReplicatedStorage,
        ReplicatedStorage:FindFirstChild("rEvents"),
        ReplicatedStorage:FindFirstChild("Events"),
        ReplicatedStorage:FindFirstChild("Remotes"),
        Player.Character,
    }
    for _, container in ipairs(search) do
        if container then
            for _, obj in ipairs(container:GetDescendants()) do
                if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                    local n = Utils.lower(obj.Name)
                    if n:find("rebirth") or n:find("renasc") or n:find("prestige") then
                        table.insert(AutoRebirth.remotes, obj)
                    end
                end
            end
        end
    end
    if #AutoRebirth.remotes > 0 then
        return AutoRebirth.remotes[1], AutoRebirth.remotes[1].Name
    end
    return nil, nil
end

function AutoRebirth.tryFire()
    local remote, name = AutoRebirth.findRebirthRemote()
    local fired = 0
    if remote then
        pcall(function()
            if remote:IsA("RemoteFunction") then
                remote:InvokeServer("rebirthRequest")
            else
                remote:FireServer("rebirthRequest")
            end
        end)
        fired = fired + 1
    end
    local confirmPaths = {
        {"rebirthMenu", "confirmButton"},
        {"RebirthMenu", "ConfirmButton"},
    }
    for _, path in ipairs(confirmPaths) do
        local pg = Player:FindFirstChild("PlayerGui")
        if pg then
            local cur = pg
            for _, n in ipairs(path) do cur = cur and cur:FindFirstChild(n) end
            if cur then Utils.signalClick(cur); fired = fired + 1 end
        end
    end
    if fired > 0 then
        AutoRebirth.status = "Disparado (" .. (name or "?") .. ")"
        return true
    else
        AutoRebirth.status = "Remote nao encontrado"
        return false
    end
end

task.spawn(function()
    while Utils.alive() do
        if AutoRebirth.active then
            local now = tick()
            if now - AutoRebirth.lastFire >= CFG.REBIRTH_RATE then
                AutoRebirth.lastFire = now
                pcall(AutoRebirth.tryFire)
            end
        end
        task.wait(0.2)
    end
end)

--==========================================================
-- 10. AUTO SIZE 2 / 10
--==========================================================
function AutoSize2.apply()
    local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
    local remote = rEvents and rEvents:FindFirstChild("changeSpeedSizeRemote")
    if remote and remote:IsA("RemoteFunction") then
        local ok, err = pcall(function()
            remote:InvokeServer("changeSize", 2)
        end)
        if ok then AutoSize2.lastError = nil; return true
        else AutoSize2.lastError = tostring(err); return false end
    else
        AutoSize2.lastError = "Remote nao encontrado"
        return false
    end
end

function AutoSize10.apply()
    local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
    local remote = rEvents and rEvents:FindFirstChild("changeSpeedSizeRemote")
    if remote and remote:IsA("RemoteFunction") then
        local ok, err = pcall(function()
            remote:InvokeServer("changeSize", 10)
        end)
        if ok then AutoSize10.lastError = nil; return true
        else AutoSize10.lastError = tostring(err); return false end
    else
        AutoSize10.lastError = "Remote nao encontrado"
        return false
    end
end

task.spawn(function()
    while Utils.alive() do
        pcall(function()
            if AutoSize2.active then
                local now = tick()
                if now - AutoSize2.lastApply >= CFG.AUTO_SIZE_RATE then
                    AutoSize2.lastApply = now
                    AutoSize2.apply()
                end
            end
            if AutoSize10.active then
                local now = tick()
                if now - AutoSize10.lastApply >= CFG.AUTO_SIZE_RATE then
                    AutoSize10.lastApply = now
                    AutoSize10.apply()
                end
            end
        end)
        task.wait(0.5)
    end
end)

--==========================================================
-- 11. AUTO CLAIM GIFT
--==========================================================
task.spawn(function()
    while Utils.alive() do
        if AutoClaimGift.active then
            pcall(function()
                local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
                local remote = rEvents and rEvents:FindFirstChild("freeGiftClaimRemote")
                if remote then
                    local i = AutoClaimGift.proximoGift
                    if remote:IsA("RemoteFunction") then
                        remote:InvokeServer("claimGift", i)
                    else
                        remote:FireServer("claimGift", i)
                    end
                    AutoClaimGift.total = AutoClaimGift.total + 1
                    AutoClaimGift.proximoGift = (i % CFG.GIFT_TOTAL) + 1
                end
            end)
        end
        task.wait(CFG.GIFT_INTERVAL)
    end
end)

--==========================================================
-- 12. AUTO FORTUNE
--==========================================================
task.spawn(function()
    while Utils.alive() do
        if AutoFortune.active then
            local now = tick()
            if now - AutoFortune.lastSpin >= CFG.FORTUNE_INTERVAL then
                AutoFortune.lastSpin = now
                pcall(function()
                    local rEvents = ReplicatedStorage:FindFirstChild("rEvents")
                    local remote = rEvents and rEvents:FindFirstChild("openFortuneWheelRemote")
                    if remote then
                        local chances
                        pcall(function()
                            chances = ReplicatedStorage.shared.catalogs.fortuneWheelChances["Fortune Wheel"]
                        end)
                        if chances then
                            if remote:IsA("RemoteFunction") then
                                remote:InvokeServer("openFortuneWheel", chances)
                            else
                                remote:FireServer("openFortuneWheel", chances)
                            end
                            AutoFortune.total = AutoFortune.total + 1
                        end
                    end
                end)
            end
        end
        task.wait(0.2)
    end
end)

--==========================================================
-- 13. BOSS FARM & AUTOMATION
--==========================================================
local Boss = {}
local chestCache = { chest = nil, lastCheck = 0 }
local BOSS_PROMPT_NAME = "BossChestPrompt"

function Boss.getChestPos(chest)
    if not chest then return nil end
    if chest:IsA("BasePart") then return chest.Position end
    local prompt = chest:FindFirstChild(BOSS_PROMPT_NAME, true)
    if prompt and prompt.Parent and prompt.Parent:IsA("BasePart") then
        return prompt.Parent.Position
    end
    for _, d in ipairs(chest:GetDescendants()) do
        if d:IsA("BasePart") then return d.Position end
    end
    return nil
end

function Boss.findChest()
    local now = tick()
    if now - chestCache.lastCheck < CFG.CHEST_CACHE_T
       and chestCache.chest and chestCache.chest.Parent then
        return chestCache.chest
    end
    chestCache.lastCheck = now

    local chest = workspace:FindFirstChild("BossChest")
    if chest then chestCache.chest = chest; return chest end

    for _, obj in ipairs(workspace:GetChildren()) do
        if obj:IsA("Model") and Utils.containsAny(obj.Name, KEYWORDS.CHEST) then
            if obj:FindFirstChild(BOSS_PROMPT_NAME, true) then
                chestCache.chest = obj; return obj
            end
        end
    end

    local ev = workspace:FindFirstChild("Events")
    local arena = ev and ev:FindFirstChild("BossArena")
    if arena then
        for _, obj in ipairs(arena:GetChildren()) do
            if obj:IsA("Model") and Utils.containsAny(obj.Name, KEYWORDS.CHEST) then
                if obj:FindFirstChild(BOSS_PROMPT_NAME, true) then
                    chestCache.chest = obj; return obj
                end
            end
        end
        
        for _, obj in ipairs(arena:GetDescendants()) do
            if obj:IsA("ProximityPrompt") and (obj.ActionText:lower():find("recompensa") or obj.ObjectText:lower():find("chefe") or obj.ActionText:lower():find("ver")) then
                chestCache.chest = obj.Parent
                return obj.Parent
            end
        end
    end

    chestCache.chest = nil
    return nil
end

function Boss.activatePrompt(prompt)
    if not prompt or not prompt:IsA("ProximityPrompt") then return false end
    pcall(function() prompt.Enabled = false end)
    pcall(function() prompt.HoldDuration = 0 end)
    pcall(function() prompt.MaxActivationDistance = 999 end)
    pcall(function() prompt.RequiresLineOfSight = false end)
    if type(fireproximityprompt) == "function" then
        pcall(fireproximityprompt, prompt, 0)
    end
    pcall(function() prompt:InputHoldBegin() end)
    task.wait(0.01)
    pcall(function() prompt:InputHoldEnd()   end)
    pcall(function() prompt.Triggered:Fire(Player) end)
    return true
end

function Boss.showClickVisual(cx, cy)
    pcall(function()
        local clickGui = PARENT_GUI:FindFirstChild("PS_ClickVisual")
        if not clickGui then
            clickGui = Utils.create("ScreenGui", {
                Name = "PS_ClickVisual",
                IgnoreGuiInset = true, ResetOnSpawn = false,
                ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
            }, PARENT_GUI)
        end
        local circle = Utils.create("Frame", {
            Position = UDim2.new(0, cx - 20, 0, cy - 20),
            Size = UDim2.new(0, 40, 0, 40),
            BackgroundColor3 = Color3.fromRGB(255, 255, 255),
            BackgroundTransparency = 0.5,
            BorderSizePixel = 0, ZIndex = 999,
        }, clickGui)
        Utils.create("UICorner", {CornerRadius = UDim.new(1,0)}, circle)
        local stroke = Utils.create("UIStroke", {
            Color = Color3.fromRGB(255, 255, 255),
            Thickness = 2, Transparency = 0,
        }, circle)
        TweenService:Create(circle, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 80, 0, 80),
            Position = UDim2.new(0, cx - 40, 0, cy - 40),
            BackgroundTransparency = 1,
        }):Play()
        TweenService:Create(stroke, TweenInfo.new(0.4), {Transparency = 1, Thickness = 4}):Play()
        task.delay(0.5, function() pcall(function() circle:Destroy() end) end)
    end)
end

function Boss.coletarBau(chest)
    if not chest then return end

    local prompt = chest:FindFirstChild(BOSS_PROMPT_NAME, true)
    if not prompt then
        for _, d in ipairs(chest:GetDescendants()) do
            if d:IsA("ProximityPrompt") then prompt = d; break end
        end
    end
    if prompt then
        pcall(function() Boss.activatePrompt(prompt) end)
        task.wait(0.1)
    end

    pcall(function()
        VirtualInputManager:SendKeyEvent(true,  Enum.KeyCode.E, false, game)
        task.wait(0.03)
        VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game)
    end)
    task.wait(0.05)
    pcall(function()
        VirtualInputManager:SendKeyEvent(true,  Enum.KeyCode.E, false, game)
        task.wait(0.03)
        VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game)
    end)

    local hrp = Player.Character and Player.Character:FindFirstChild("HumanoidRootPart")
    if hrp then
        if type(firetouchinterest) == "function" then
            for _, d in ipairs(chest:GetDescendants()) do
                if d:IsA("BasePart") then
                    pcall(function()
                        firetouchinterest(d, hrp, 0)
                        task.wait(0.01)
                        firetouchinterest(d, hrp, 1)
                    end)
                end
            end
        end
        
        if type(fireclickdetector) == "function" then
            for _, cd in ipairs(chest:GetDescendants()) do
                if cd:IsA("ClickDetector") then
                    pcall(function() fireclickdetector(cd) end)
                    task.wait(0.01)
                    pcall(function() fireclickdetector(cd, 0) end)
                end
            end
        end
    end

    if Utils.isMobile() then
        pcall(function()
            local cam = workspace.CurrentCamera
            local vp  = cam and cam.ViewportSize or Vector2.new(800, 600)
            local cx  = math.floor(vp.X / 2)
            local cy  = math.floor(vp.Y / 2)
            local cpos = Boss.getChestPos(chest)
            if cam and cpos then
                local sp, onScreen = cam:WorldToViewportPoint(cpos)
                if onScreen then
                    cx = math.floor(sp.X); cy = math.floor(sp.Y)
                end
            end
            Boss.showClickVisual(cx, cy)
        end)
    end
end

function Boss.getHealth(m)    return m and (m:GetAttribute("Health") or m:GetAttribute("BossHealth")) end
function Boss.getMaxHealth(m) return m and (m:GetAttribute("MaxHealth") or m:GetAttribute("BossMaxHealth")) end
function Boss.isEmerging(m)   return m and (m:GetAttribute("BossEmergeing") or m:GetAttribute("BossEmerging")) or false end

function Boss.isAlive(slot)
    if not slot then return false end
    local hp, maxHp, em = Boss.getHealth(slot), Boss.getMaxHealth(slot), Boss.isEmerging(slot)
    if hp and maxHp and hp > 0 and maxHp > 0 and not em then return true end
    local hum = slot:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

function Boss.getBodyPart(m)
    if not m then return nil end
    local b = m:FindFirstChild("Boss")
    local hb = m:FindFirstChild("BossDamageHitbox")
    if b and b:IsA("BasePart") then return b end
    if hb and hb:IsA("BasePart") then return hb end
    local hrp = m:FindFirstChild("HumanoidRootPart")
    if hrp and hrp:IsA("BasePart") then return hrp end
    return nil
end

function Boss.getHeadPart(m)
    if not m then return nil end
    local head = m:FindFirstChild("Head")
    if head and head:IsA("BasePart") then return head end
    local bestHead, bestHeadY = nil, -math.huge
    for _, d in ipairs(m:GetDescendants()) do
        if d:IsA("BasePart") then
            local n = Utils.lower(d.Name)
            if n:find("head") and not n:find("overhead") then
                if d.Position.Y > bestHeadY then
                    bestHeadY = d.Position.Y; bestHead = d
                end
            end
        end
    end
    if bestHead then return bestHead end
    local highest, highestY = nil, -math.huge
    for _, d in ipairs(m:GetDescendants()) do
        if d:IsA("BasePart") and d.Transparency < 1 and d.CanCollide ~= false then
            if d.Position.Y > highestY then
                highestY = d.Position.Y; highest = d
            end
        end
    end
    if highest then return highest end
    local upper = m:FindFirstChild("UpperTorso"); if upper then return upper end
    local torso = m:FindFirstChild("Torso"); if torso then return torso end
    local boss = m:FindFirstChild("Boss"); if boss then return boss end
    local hb = m:FindFirstChild("BossDamageHitbox"); if hb then return hb end
    local hrp = m:FindFirstChild("HumanoidRootPart"); if hrp then return hrp end
    return nil
end

local arenaCache = { model = nil, lastCheck = 0 }

function Boss.findArena()
    local now = tick()
    if arenaCache.model and arenaCache.model.Parent and (now - arenaCache.lastCheck) < 2 then
        return arenaCache.model
    end
    arenaCache.lastCheck = now
    arenaCache.model = nil

    local ev = workspace:FindFirstChild("Events")
    local arena = ev and ev:FindFirstChild("BossArena")
    if arena then arenaCache.model = arena; return arena end

    arena = workspace:FindFirstChild("BossArena")
    if arena then arenaCache.model = arena; return arena end

    return nil
end

function Boss.findAny()
    local arena = Boss.findArena()
    if not arena then return nil end

    for _, child in ipairs(arena:GetChildren()) do
        if child:IsA("Model") then
            if Boss.isAlive(child) then
                local bp = Boss.getBodyPart(child)
                if bp then 
                    return bp, child.Name, arena 
                end
            end
        end
    end

    return nil
end

function Boss.getPunchTool()
    local char = Player.Character
    if not char then return nil end
    local bp = Player:FindFirstChild("Backpack")
    for _, t in ipairs(char:GetChildren()) do
        if t:IsA("Tool") and Anim.isPunchTool(t) then return t end
    end
    if bp then
        for _, t in ipairs(bp:GetChildren()) do
            if t:IsA("Tool") and Anim.isPunchTool(t) then return t end
        end
    end
    return nil
end

function Boss.createHitbox()
    if State.autoBoss.hitboxPart and State.autoBoss.hitboxPart.Parent then
        return State.autoBoss.hitboxPart
    end
    local hrp = Utils.getHRP()
    if not hrp then return nil end
    local size = 10
    local hb = Instance.new("Part")
    hb.Name = "PS_BossHitbox"
    hb.Size = Vector3.new(size, size, size)
    hb.Transparency = 1
    hb.CanCollide = false
    hb.CanTouch = true
    hb.CanQuery = true
    hb.Massless = true
    hb.Anchored = false
    hb.CFrame = CFrame.new(hrp.Position + Vector3.new(0, -CFG.HITBOX_DOWN_OFFSET, 0))
    hb.Parent = Player.Character
    local weld = Instance.new("WeldConstraint")
    weld.Part0, weld.Part1 = hrp, hb
    weld.Parent = hb
    State.autoBoss.hitboxPart = hb
    return hb
end

function Boss.destroyHitbox()
    if State.autoBoss.hitboxPart then
        pcall(function() State.autoBoss.hitboxPart:Destroy() end)
        State.autoBoss.hitboxPart = nil
    end
end

-- ============ LOOP PRINCIPAL BOSS NORMAL ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local b = State.autoBoss
            if b.active then
                if not (b.hitboxPart and b.hitboxPart.Parent) then Boss.createHitbox() end
                local myHrp = Utils.getHRP()

                if b.mode == "idle" and myHrp then
                    b.savedArenaPos = BOSS_WAIT_POS
                    b.hasTeleported = true
                    b.mode = "waiting_spawn"
                    myHrp.CFrame = CFrame.new(BOSS_WAIT_POS)
                    myHrp.Velocity = Vector3.zero
                    myHrp.AssemblyLinearVelocity = Vector3.zero
                end

                if b.mode == "waiting_spawn" or b.mode == "fighting" then
                    local body, slot, arena = Boss.findAny()
                    if body and arena then
                        if b.mode == "waiting_spawn" or b.target ~= body then
                            b.target = body
                            b.slotName = slot
                            b.mode = "fighting"
                            b.currentBossName = slot or "Boss"
                            if b.lastBossHp == 0 then
                                b.totalDamage = 0
                            end
                        end
                    elseif b.mode == "fighting" then
                        b.mode = "waiting_chest"
                        b.chestWaitStart = tick()
                        b.target = nil
                        b.lastBossHp = 0
                    end
                end

                if b.mode == "waiting_chest" then
                    local chest = Boss.findChest()
                    if chest then
                        b.target = chest
                        b.mode = "opening_chest"
                        b.chestStartTime = tick()
                        b.chestTPDone = false
                        b.chestCollected = false
                    elseif tick() - (b.chestWaitStart or 0) > 15 then
                        b.mode, b.target = "waiting_spawn", nil
                    end
                end

                if b.mode == "opening_chest" then
                    local chest = b.target
                    if not chest or not chest.Parent then
                        b.mode, b.target = "waiting_spawn", nil
                        b.chestTPDone = false
                        b.chestCollected = false
                        chestCache.chest = nil
                    elseif tick() - (b.chestStartTime or 0) > CFG.CHEST_MAX_TIME then
                        b.mode, b.target = "waiting_spawn", nil
                        b.chestTPDone = false
                        b.chestCollected = false
                        chestCache.chest = nil
                    end
                end

                if b.mode == "fighting" and b.target and b.target.Parent then
                    local hp    = Boss.getHealth(b.target.Parent) or 0
                    local maxHp = Boss.getMaxHealth(b.target.Parent) or 0
                    local hum = b.target.Parent:FindFirstChildOfClass("Humanoid")
                    if hum then hp, maxHp = hum.Health, hum.MaxHealth end

                    if b.lastBossHp == 0 or hp > b.lastBossHp then
                        b.totalDamage = 0
                    elseif b.lastBossHp > hp then
                        b.totalDamage = b.totalDamage + (b.lastBossHp - hp)
                    end
                    b.lastBossHp = hp
                    b.currentHp, b.maxHp = hp, maxHp
                end
            else
                if State.autoBoss.hitboxPart then Boss.destroyHitbox() end
                State.autoBoss.currentHp = 0
                State.autoBoss.maxHp = 0
                State.autoBoss.currentBossName = ""
                State.autoBoss.savedArenaPos = nil
            end
        end)
        task.wait(0.1)
    end
end)

-- ============ LABEL HP BOSS NORMAL ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            if BossHpLabel and BossHpLabel.Parent then
                local b = State.autoBoss
                if b.active then
                    if b.mode == "fighting" and b.target and b.target.Parent then
                        local hp    = Boss.getHealth(b.target.Parent) or 0
                        local maxHp = Boss.getMaxHealth(b.target.Parent) or 0
                        local hum = b.target.Parent:FindFirstChildOfClass("Humanoid")
                        if hum then hp, maxHp = hum.Health, hum.MaxHealth end
                        local pct = maxHp > 0 and math.floor((hp / maxHp) * 100) or 0
                        local name = b.currentBossName ~= "" and b.currentBossName or "Boss"
                        BossHpLabel.Text = string.format("[%s]  HP: %d / %d  (%d%%)", name, hp, maxHp, pct)
                        BossHpLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
                        if BossHpBarFill then
                            BossHpBarFill.Size = UDim2.new(math.clamp(hp/math.max(maxHp,1),0,1),0,1,0)
                            if pct > 60 then BossHpBarFill.BackgroundColor3 = Color3.fromRGB(0,200,80)
                            elseif pct > 30 then BossHpBarFill.BackgroundColor3 = Color3.fromRGB(255,200,0)
                            else BossHpBarFill.BackgroundColor3 = Color3.fromRGB(255,50,50) end
                        end
                    elseif b.mode == "opening_chest" then
                        BossHpLabel.Text = "Resgatando BossChest..."
                        BossHpLabel.TextColor3 = Color3.fromRGB(255, 200, 0)
                        if BossHpBarFill then BossHpBarFill.Size = UDim2.new(0,0,1,0) end
                    else
                        BossHpLabel.Text = "Procurando boss..."
                        BossHpLabel.TextColor3 = Color3.fromRGB(150,150,150)
                        if BossHpBarFill then BossHpBarFill.Size = UDim2.new(0,0,1,0) end
                    end

                    if State.autoBoss.damageDisplay and State.autoBoss.damageDisplay.Parent then
                        State.autoBoss.damageDisplay.Text = "DANO TOTAL: " .. Utils.formatNumber(b.totalDamage or 0)
                        if (b.totalDamage or 0) > 1000000 then
                            State.autoBoss.damageDisplay.TextColor3 = Color3.fromRGB(255, 215, 0)
                        elseif (b.totalDamage or 0) > 100000 then
                            State.autoBoss.damageDisplay.TextColor3 = Color3.fromRGB(255, 140, 0)
                        else
                            State.autoBoss.damageDisplay.TextColor3 = Color3.fromRGB(255, 80, 80)
                        end
                    end
                else
                    BossHpLabel.Text = "Auto Boss: OFF"
                    BossHpLabel.TextColor3 = Color3.fromRGB(150,150,150)
                    if BossHpBarFill then BossHpBarFill.Size = UDim2.new(0,0,1,0) end

                    if State.autoBoss.damageDisplay and State.autoBoss.damageDisplay.Parent then
                        State.autoBoss.damageDisplay.Text = "DANO TOTAL: 0"
                        State.autoBoss.damageDisplay.TextColor3 = Color3.fromRGB(150, 150, 150)
                    end
                end
            end
        end)
        task.wait(CFG.HP_UPDATE_RATE)
    end
end)

-- ============ SEGUE O BOSS NORMAL (throttled) ============
local normalBossStepAcc = 0
RunService.RenderStepped:Connect(function()
    normalBossStepAcc = normalBossStepAcc + 1
    if normalBossStepAcc < 3 then return end
    normalBossStepAcc = 0

    pcall(function()
        if not Utils.alive() then return end
        local b = State.autoBoss
        if not b.active then return end
        if State.lockPos.active then return end
        local hrp = Utils.getHRP()
        if not hrp then return end

        if b.mode == "fighting" and b.target and b.target.Parent then
            local headPart = Boss.getHeadPart(b.target.Parent)
            if headPart then
                local headHalfY   = headPart.Size.Y * 0.5
                local playerHalfY = hrp.Size.Y * 0.5
                local gap = 0.1
                local offsetY = headHalfY + playerHalfY + gap
                offsetY = offsetY + (State.autoBoss.offsetY or 0)

                local targetPos = headPart.Position + Vector3.new(0, offsetY, 0)
                local flatCFrame = CFrame.new(targetPos) * CFrame.Angles(math.rad(-90), 0, 0)
                hrp.CFrame = flatCFrame
                hrp.Velocity = Vector3.zero
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.RotVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
            end
            return
        end

        if b.mode == "opening_chest" and b.target and b.target.Parent then
            local chestPos = Boss.getChestPos(b.target)
            if chestPos then
                if not b.chestTPDone then
                    b.chestTPDone = true
                    hrp.CFrame = CFrame.new(chestPos + Vector3.new(0, CFG.CHEST_HEAD_OFFSET, 0))
                    hrp.Velocity = Vector3.zero
                    hrp.AssemblyLinearVelocity = Vector3.zero
                else
                    local alvo = chestPos + Vector3.new(0, CFG.CHEST_HEAD_OFFSET, 0)
                    local dist = (hrp.Position - alvo).Magnitude
                    if dist > 3 then
                        hrp.CFrame = CFrame.new(alvo)
                        hrp.Velocity = Vector3.zero
                        hrp.AssemblyLinearVelocity = Vector3.zero
                    end
                end
            end
            return
        end

        if b.savedArenaPos and (b.mode == "waiting_spawn" or b.mode == "waiting_chest") then
            local dist = (hrp.Position - b.savedArenaPos).Magnitude
            if dist > 5 then
                hrp.CFrame = CFrame.new(b.savedArenaPos)
                hrp.Velocity = Vector3.zero
                hrp.AssemblyLinearVelocity = Vector3.zero
            end
        end
    end)
end)

-- ============ ATACA O BOSS NORMAL ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local b = State.autoBoss
            if b.active and b.mode == "fighting" then
                local char = Player.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                local punch = Boss.getPunchTool()
                if punch and hum then
                    local cur = char:FindFirstChildOfClass("Tool")
                    if cur and cur ~= punch then
                        hum:UnequipTools()
                        task.wait(0.02)
                    end
                    if punch.Parent ~= char then
                        hum:EquipTool(punch)
                        task.wait(0.02)
                    end
                    for _, track in ipairs(hum:GetPlayingAnimationTracks()) do
                        local a = track.Animation
                        local nm = a and a.Name or ""
                        local id = a and tostring(a.AnimationId) or ""
                        if not Anim.isMovementAnim(nm, id) then
                            track:AdjustSpeed(CFG.PUNCH_FAST_SPEED)
                        end
                    end
                    punch:Activate()
                end
            end
        end)
        task.wait(CFG.PUNCH_RATE)
    end
end)

-- ============ ABRE O CHEST NORMAL ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local b = State.autoBoss
            if b.active and b.mode == "opening_chest"
               and b.target and b.target.Parent then
                local chest = b.target
                if not b.chestCollected then
                    b.chestCollected = true
                    Boss.coletarBau(chest)
                    task.wait(0.5)
                    if not chest.Parent then
                        b.mode, b.target = "waiting_spawn", nil
                        b.chestCollected = false
                        b.chestTPDone = false
                        chestCache.chest = nil
                    else
                        Boss.coletarBau(chest)
                        task.wait(0.5)
                        b.mode, b.target = "waiting_spawn", nil
                        b.chestCollected = false
                        b.chestTPDone = false
                        chestCache.chest = nil
                    end
                end
            end
        end)
        task.wait(0.1)
    end
end)

--==========================================================
-- 13B. AUTO BOSS E FARM
--==========================================================
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local bFarm = State.autoBossFarm
            local bBoss = State.autoBoss
            local char = Player.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            
            if bFarm.active and hum then
                if bBoss.mode == "waiting_spawn" or bBoss.mode == "idle" then
                    local cur = char:FindFirstChildOfClass("Tool")
                    local isW = cur and not Anim.isPunchTool(cur) and 
                        (Utils.lower(cur.Name):find("weight") or Utils.lower(cur.Name):find("peso"))
                    
                    if not isW then
                        local bp = Player:FindFirstChild("Backpack")
                        if bp then
                            for _, t in ipairs(bp:GetChildren()) do
                                if t:IsA("Tool") then
                                    local n = Utils.lower(t.Name)
                                    if n:find("weight") or n:find("peso") then
                                        hum:EquipTool(t)
                                        break
                                    end
                                end
                            end
                        end
                    else
                        cur:Activate()
                    end
                elseif bBoss.mode == "fighting" then
                    local punch = Boss.getPunchTool()
                    local cur = char:FindFirstChildOfClass("Tool")
                    if cur and cur ~= punch then
                        hum:UnequipTools()
                        task.wait(0.05)
                    end
                    if punch and punch.Parent ~= char then
                        hum:EquipTool(punch)
                    end
                end
            end
        end)
        task.wait(0.1)
    end
end)

--==========================================================
-- 13C. HALLOWEEN BOSS
--==========================================================
local HWBoss = {}

function HWBoss.getHealth(m)    return m and (m:GetAttribute("Health") or m:GetAttribute("BossHealth")) end
function HWBoss.getMaxHealth(m) return m and (m:GetAttribute("MaxHealth") or m:GetAttribute("BossMaxHealth")) end
function HWBoss.isEmerging(m)   return m and (m:GetAttribute("BossEmergeing") or m:GetAttribute("BossEmerging")) or false end

function HWBoss.isAlive(model)
    if not model then return false end
    local hp = HWBoss.getHealth(model)
    local maxHp = HWBoss.getMaxHealth(model)
    local em = HWBoss.isEmerging(model)
    if hp and maxHp and hp > 0 and maxHp > 0 and not em then return true end
    local hum = model:FindFirstChildOfClass("Humanoid")
    if hum and hum.Health > 0 then return true end
    local seq = model:GetAttribute("BossTargetSequence")
    if seq and seq > 0 then return true end
    return false
end

function HWBoss.find(folderName)
    if not folderName then return nil end
    local direct = workspace:FindFirstChild(folderName)
    if direct and direct:IsA("Model") then return direct end
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj.Name == folderName and obj:IsA("Model") then return obj end
    end
    return nil
end

function HWBoss.getHeadPart(model)
    if not model then return nil end
    local head = model:FindFirstChild("Head", true)
    if head and head:IsA("BasePart") then return head end

    local namedHead, namedHeadY = nil, -math.huge
    for _, d in ipairs(model:GetDescendants()) do
        if d:IsA("BasePart") then
            local n = Utils.lower(d.Name)
            if (n:find("head") or n:find("skull") or n:find("cranium"))
                and not n:find("overhead") and not n:find("hitbox") then
                if d.Position.Y > namedHeadY then
                    namedHeadY, namedHead = d.Position.Y, d
                end
            end
        end
    end
    if namedHead then return namedHead end

    local meshParts = {}
    for _, d in ipairs(model:GetChildren()) do
        if d:IsA("MeshPart") and d.Transparency < 1 then
            table.insert(meshParts, d)
        end
    end
    if #meshParts > 0 then
        local highest, highestY = meshParts[1], meshParts[1].Position.Y
        for _, mp in ipairs(meshParts) do
            if mp.Position.Y > highestY then
                highestY, highest = mp.Position.Y, mp
            end
        end
        return highest
    end

    local hitbox = model:FindFirstChild("BossDamageHitbox")
    if hitbox and hitbox:IsA("BasePart") then return hitbox end

    local highest, highestY = nil, -math.huge
    for _, d in ipairs(model:GetDescendants()) do
        if d:IsA("BasePart") and d.Transparency < 1 then
            if d.Position.Y > highestY then
                highestY, highest = d.Position.Y, d
            end
        end
    end
    return highest
end

function HWBoss.createHitbox(boss)
    if boss.hitbox and boss.hitbox.Parent then return boss.hitbox end
    local hrp = Utils.getHRP()
    if not hrp then return nil end
    local hb = Instance.new("Part")
    hb.Name = "PS_HW_" .. boss.id .. "_Hitbox"
    hb.Size = Vector3.new(CFG.HITBOX_SIZE, CFG.HITBOX_SIZE, CFG.HITBOX_SIZE)
    hb.Transparency = 1
    hb.CanCollide = false
    hb.CanTouch = true
    hb.CanQuery = true
    hb.Massless = true
    hb.Anchored = false
    hb.CFrame = CFrame.new(hrp.Position + Vector3.new(0, -CFG.HITBOX_DOWN_OFFSET, 0))
    hb.Parent = Player.Character
    local weld = Instance.new("WeldConstraint")
    weld.Part0, weld.Part1 = hrp, hb
    weld.Parent = hb
    boss.hitbox = hb
    return hb
end

function HWBoss.destroyHitbox(boss)
    if boss.hitbox then
        pcall(function() boss.hitbox:Destroy() end)
        boss.hitbox = nil
    end
end

function HWBoss.desativarTodos(exceptBoss)
    for _, b in ipairs(HW_BOSSES) do
        if b ~= exceptBoss and b.active then
            b.active = false
            HWBoss.destroyHitbox(b)
        end
    end
end

-- ============ LOOP DE BUSCA HW ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            for _, boss in ipairs(HW_BOSSES) do
                if boss.active then
                    local model = HWBoss.find(boss.folder)
                    if model and HWBoss.isAlive(model) then
                        local head = HWBoss.getHeadPart(model)
                        if head then
                            boss.target = head
                            boss.mode = "teleportando"
                            boss.hp = HWBoss.getHealth(model) or 0
                            boss.maxHp = HWBoss.getMaxHealth(model) or 0
                            local hum = model:FindFirstChildOfClass("Humanoid")
                            if hum then boss.hp, boss.maxHp = hum.Health, hum.MaxHealth end
                        else
                            boss.target = nil
                            boss.mode = "sem_part"
                        end
                    else
                        boss.target = nil
                        boss.mode = model and "boss_morto" or "sem_arena"
                        boss.hp, boss.maxHp = 0, 0
                    end
                else
                    boss.target = nil
                    boss.mode = "idle"
                    boss.hp, boss.maxHp = 0, 0
                end
            end
        end)
        task.wait(CFG.SEARCH_RATE)
    end
end)

-- ============ TELEPORTE HW (throttled) ============
local hwStepAcc = 0
RunService.RenderStepped:Connect(function()
    hwStepAcc = hwStepAcc + 1
    if hwStepAcc < 3 then return end
    hwStepAcc = 0

    pcall(function()
        if not Utils.alive() then return end
        local anyActive = false
        for _, b in ipairs(HW_BOSSES) do
            if b.active then anyActive = true; break end
        end
        if not anyActive then return end

        local hrp = Utils.getHRP()
        if not hrp then return end
        if State.lockPos and State.lockPos.active then return end

        for _, boss in ipairs(HW_BOSSES) do
            if boss.active then
                HWBoss.createHitbox(boss)

                if boss.mode == "teleportando" and boss.target and boss.target.Parent then
                    local part = boss.target
                    local partHalfHeight = part.Size.Y * 0.5
                    local playerHalfHeight = hrp.Size.Y * 0.5
                    local offset = partHalfHeight + playerHalfHeight + CFG.HW_HEAD_OFFSET
                    local targetPos = part.Position + Vector3.new(0, offset + boss.offsetY, 0)

                    hrp.CFrame = CFrame.new(targetPos) * CFrame.Angles(math.rad(-90), 0, 0)
                    hrp.Velocity = Vector3.zero
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.RotVelocity = Vector3.zero
                    hrp.AssemblyAngularVelocity = Vector3.zero
                end

                if boss.mode == "boss_morto" or boss.mode == "sem_arena" then
                    local spawnPos = boss.spawnPos
                    local dist = (hrp.Position - spawnPos).Magnitude
                    if dist > 5 then
                        hrp.CFrame = CFrame.new(spawnPos)
                        hrp.Velocity = Vector3.zero
                        hrp.AssemblyLinearVelocity = Vector3.zero
                    end
                end
            else
                HWBoss.destroyHitbox(boss)
            end
        end
    end)
end)

-- ============ AUTO PUNCH HW ============
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local algumAtivo = false
            for _, b in ipairs(HW_BOSSES) do
                if b.active and b.mode == "teleportando" then
                    algumAtivo = true
                    break
                end
            end
            if algumAtivo then
                local char = Player.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hum then
                    local punch = Boss.getPunchTool()
                    if punch then
                        local cur = char:FindFirstChildOfClass("Tool")
                        if cur and cur ~= punch then
                            hum:UnequipTools()
                            task.wait(0.02)
                        end
                        if punch.Parent ~= char then
                            hum:EquipTool(punch)
                            task.wait(0.02)
                        end
                        for _, track in ipairs(hum:GetPlayingAnimationTracks()) do
                            local a = track.Animation
                            local nm = a and a.Name or ""
                            local id = a and tostring(a.AnimationId) or ""
                            if not Anim.isMovementAnim(nm, id) then
                                track:AdjustSpeed(CFG.PUNCH_FAST_SPEED)
                            end
                        end
                        punch:Activate()
                    end
                end
            end
        end)
        task.wait(CFG.PUNCH_RATE)
    end
end)

--==========================================================
-- 13D. AUTO PUNCH FAST global
--==========================================================
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local p = State.autoPunch
            if p.active then
                local char = Player.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hum then
                    local punch = Boss.getPunchTool()
                    if punch then
                        local cur = char:FindFirstChildOfClass("Tool")
                        if cur and cur ~= punch then
                            hum:UnequipTools()
                            task.wait(0.02)
                        end
                        if punch.Parent ~= char then
                            hum:EquipTool(punch)
                            task.wait(0.02)
                        end
                        for _, track in ipairs(hum:GetPlayingAnimationTracks()) do
                            local a = track.Animation
                            local nm = a and a.Name or ""
                            local id = a and tostring(a.AnimationId) or ""
                            if not Anim.isMovementAnim(nm, id) then
                                track:AdjustSpeed(CFG.PUNCH_FAST_SPEED)
                            end
                        end
                        punch:Activate()
                    end
                end
            end
        end)
        task.wait(CFG.PUNCH_RATE)
    end
end)

--==========================================================
-- 13E. HALLOWEEN PASS
--==========================================================
task.spawn(function()
    while Utils.alive() do
        if AutoClaimQuest.active then
            pcall(function()
                local remote = Utils.getHalloweenPassRemote()
                if remote then
                    local categorias = {"Daily", "Seasonal", "Weekly", "Normal", "Event", "Season"}
                    for _, cat in ipairs(categorias) do
                        if remote:IsA("RemoteFunction") then
                            remote:InvokeServer("ClaimQuest", { Category = cat })
                        else
                            remote:FireServer("ClaimQuest", { Category = cat })
                        end
                        task.wait(0.15)
                    end
                end
            end)
        end
        task.wait(2)
    end
end)

task.spawn(function()
    while Utils.alive() do
        if AutoBypass.active then
            pcall(function()
                local remote = Utils.getHalloweenPassRemote()
                if remote then
                    for tier = 1, 10 do
                        if not AutoBypass.active then break end
                        if remote:IsA("RemoteFunction") then
                            remote:InvokeServer("ClaimTier", { Tier = tier })
                        else
                            remote:FireServer("ClaimTier", { Tier = tier })
                        end
                        AutoBypass.currentTier = tier
                        task.wait(0.2)
                    end
                end
            end)
        end
        task.wait(3)
    end
end)

task.spawn(function()
    while Utils.alive() do
        if AutoPrestige.active then
            pcall(function()
                local remote = Utils.getHalloweenPassRemote()
                if remote then
                    local now = tick()
                    if now - AutoPrestige.lastFire >= 2 then
                        AutoPrestige.lastFire = now
                        if remote:IsA("RemoteFunction") then
                            remote:InvokeServer("Prestige", {})
                        else
                            remote:FireServer("Prestige", {})
                        end
                    end
                end
            end)
        end
        task.wait(0.5)
    end
end)

--==========================================================
-- 14. UI HELPERS
--==========================================================
local UI = {}
UI.Theme = {
    Container      = Color3.fromRGB(18, 17, 28),
    Card           = Color3.fromRGB(30, 28, 45),
    CardHover      = Color3.fromRGB(40, 37, 58),
    CardDark       = Color3.fromRGB(22, 21, 32),
    Border         = Color3.fromRGB(58, 52, 82),
    BorderSoft     = Color3.fromRGB(44, 40, 64),
    Accent         = Color3.fromRGB(168, 85, 247),
    Accent2        = Color3.fromRGB(139, 92, 246),
    AccentGlow     = Color3.fromRGB(192, 132, 252),
    Text           = Color3.fromRGB(240, 238, 250),
    TextMuted      = Color3.fromRGB(160, 156, 185),
    TextDark       = Color3.fromRGB(120, 116, 145),
    Success        = Color3.fromRGB(52, 211, 153),
    Green          = Color3.fromRGB(34, 197, 94),
    Danger         = Color3.fromRGB(248, 113, 113),
    Fast           = Color3.fromRGB(248, 113, 113),
    FastBlue       = Color3.fromRGB(96, 165, 250),
    Rebirth        = Color3.fromRGB(251, 146, 60),
    Boss           = Color3.fromRGB(236, 72, 153),
    Halloween      = Color3.fromRGB(251, 146, 60),
    Gold           = Color3.fromRGB(250, 204, 21),
    AntiAfk        = Color3.fromRGB(45, 212, 191),
    Welcome        = Color3.fromRGB(125, 211, 252),
    Discord        = Color3.fromRGB(88, 101, 242),
    DiscordHi      = Color3.fromRGB(112, 125, 255),
}

UI.RADIUS      = UDim.new(0, 12)
UI.RADIUS_SM   = UDim.new(0, 8)
UI.RADIUS_PILL = UDim.new(1, 0)

function UI.stroke(parent, color, thickness, transparency)
    return Utils.create("UIStroke", {
        Color = color or UI.Theme.Border,
        Thickness = thickness or 1,
        Transparency = transparency or 0.3,
        ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
    }, parent)
end

function UI.corner(parent, radius)
    return Utils.create("UICorner", {
        CornerRadius = radius or UI.RADIUS,
    }, parent)
end

function UI.createTitleBar(topBar, subtitle)
    local T = UI.Theme
    local frame = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, -70, 1, 0),
    }, topBar)

    local dot = Utils.create("Frame", {
        BackgroundColor3 = T.Accent,
        BorderSizePixel = 0,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 0, 0.5, 0),
        Size = UDim2.new(0, 8, 0, 8),
    }, frame)
    UI.corner(dot, UDim.new(1, 0))
    Utils.create("UIStroke", {
        Color = T.AccentGlow, Thickness = 1, Transparency = 0.4,
    }, dot)

    Utils.create("TextLabel", {
        Font = CFG.FONT, Text = CFG.TITLE,
        TextColor3 = T.Text, TextSize = subtitle or 15,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 16, 0, 0),
        Size = UDim2.new(0, 140, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
    }, frame)

    Utils.create("Frame", {
        BackgroundColor3 = T.Border,
        BorderSizePixel = 0,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 162, 0.5, 0),
        Size = UDim2.new(0, 1, 0, 14),
    }, frame)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = CFG.TITLE_WELCOME,
        TextColor3 = T.Welcome, TextSize = (subtitle or 15) - 2,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 172, 0, 0),
        Size = UDim2.new(1, -172, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
    }, frame)

    return frame
end

function UI.createWindowButtons(topBar, onClose, onMinimize, onRestore)
    local T = UI.Theme
    local function makeBtn(text, hoverColor, xPos)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = text,
            TextColor3 = T.TextMuted, TextSize = 14,
            BackgroundColor3 = T.CardDark,
            BorderSizePixel = 0,
            AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.new(1, xPos, 0.5, 0),
            Size = UDim2.new(0, 26, 0, 26),
            AutoButtonColor = false,
        }, topBar)
        UI.corner(b, UDim.new(1, 0))
        local s = Utils.create("UIStroke", {
            Color = T.Border, Thickness = 1, Transparency = 0.4,
        }, b)
        b.MouseEnter:Connect(function()
            TweenService:Create(b, TweenInfo.new(0.15), {BackgroundColor3 = hoverColor}):Play()
            TweenService:Create(s, TweenInfo.new(0.15), {Transparency = 0}):Play()
        end)
        b.MouseLeave:Connect(function()
            TweenService:Create(b, TweenInfo.new(0.15), {BackgroundColor3 = T.CardDark}):Play()
            TweenService:Create(s, TweenInfo.new(0.15), {Transparency = 0.4}):Play()
        end)
        return b
    end

    local close = makeBtn("✕", Color3.fromRGB(190, 50, 60), 0)
    close.Activated:Connect(function() if onClose then onClose() end end)

    local min = makeBtn("—", T.CardHover, -32)
    min.Activated:Connect(function() if onMinimize then onMinimize() end end)

    return { Close = close, Min = min }
end

--==========================================================
-- 15. BOTÃO FLUTUANTE
--==========================================================
local FloatingBtnGui = nil

local function createFloatingButton()
    if not Utils.alive() then return end
    if FloatingBtnGui then FloatingBtnGui:Destroy() end
    local T = UI.Theme
    FloatingBtnGui = Utils.create("ScreenGui", {
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true, Name = "PS_FloatingBtn",
        ResetOnSpawn = false,
    }, PARENT_GUI)

    local btn = Utils.create("ImageButton", {
        Size = UDim2.new(0, 0, 0, 0),
        Position = UDim2.new(1, -90, 0, 90),
        BackgroundColor3 = T.Container,
        BorderSizePixel = 0,
        Image = CFG.FLOAT_ICON_ID,
        ImageColor3 = Color3.fromRGB(255, 255, 255),
        ImageTransparency = 0,
        ScaleType = Enum.ScaleType.Fit,
        AutoButtonColor = false,
    }, FloatingBtnGui)
    UI.corner(btn, UDim.new(0, 16))
    local stroke = Utils.create("UIStroke", {
        Color = T.Accent, Thickness = 2, Transparency = 0.1,
    }, btn)

    TweenService:Create(btn, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 72, 0, 72)
    }):Play()

    local dragging = false
    local dragStart, startPos
    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = btn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            btn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    btn.MouseEnter:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = T.Card}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.15), {Transparency = 0}):Play()
    end)
    btn.MouseLeave:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = T.Container}):Play()
        TweenService:Create(stroke, TweenInfo.new(0.15), {Transparency = 0.1}):Play()
    end)

    task.spawn(function()
        while FloatingBtnGui and FloatingBtnGui.Parent and Utils.alive() do
            TweenService:Create(stroke, TweenInfo.new(0.9,
                Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                {Transparency = 0.6}):Play()
            task.wait(0.9)
            if not (FloatingBtnGui and FloatingBtnGui.Parent) then break end
            TweenService:Create(stroke, TweenInfo.new(0.9,
                Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                {Transparency = 0.1}):Play()
            task.wait(0.9)
        end
    end)

    btn.Activated:Connect(function()
        if not Utils.alive() then return end
        TweenService:Create(btn, TweenInfo.new(0.25, Enum.EasingStyle.Quad), {Size = UDim2.new(0, 0, 0, 0)}):Play()
        task.wait(0.25)
        
        if FloatingBtnGui then FloatingBtnGui:Destroy(); FloatingBtnGui = nil end
        if not Utils.alive() then return end
        if State.lastWindow == "fast" then openAutoFastWindow()
        else openMainHub() end
    end)
    return btn
end

--==========================================================
-- 16. LAUNCHER (COM ANIMAÇÃO + X DESLIGA SCRIPT)
--==========================================================
local MainGui, AutoFastGui

local function shutdownScript()
    TOKEN.dead = true

    State.autoBoss.active = false
    State.autoBossFarm.active = false
    State.autoPunch.active = false
    State.autoTP.active = false
    State.lockPos.active = false
    State.antiAfk.active = false
    State.autoMedalShop.active = false
    if AutoFast then AutoFast.active = false; pcall(AutoFast.stopRepLoop) end
    if AutoRebirth then AutoRebirth.active = false end
    if AutoTP then AutoTP.active = false end
    if AutoSize2 then AutoSize2.active = false end
    if AutoSize10 then AutoSize10.active = false end
    if AutoClaimGift then AutoClaimGift.active = false end
    if AutoFortune then AutoFortune.active = false end
    if AutoClaimQuest then AutoClaimQuest.active = false end
    if AutoBypass then AutoBypass.active = false end
    if AutoPrestige then AutoPrestige.active = false end

    for _, tool in pairs(State.tools) do
        tool.auto = false
        tool.fast = false
    end

    for _, boss in ipairs(HW_BOSSES) do
        boss.active = false
        boss.target = nil
        boss.mode = "idle"
        if boss.hitbox then pcall(function() boss.hitbox:Destroy() end); boss.hitbox = nil end
    end

    if State.autoBoss.hitboxPart then
        pcall(function() State.autoBoss.hitboxPart:Destroy() end)
        State.autoBoss.hitboxPart = nil
    end

    pcall(function() cleanupGuis(PARENT_GUI) end)
    pcall(function() cleanupGuis(Player:FindFirstChild("PlayerGui")) end)
    pcall(function() if gethui then cleanupGuis(gethui()) end end)
    pcall(function() cleanupGuis(game:GetService("CoreGui")) end)

    MainGui = nil
    AutoFastGui = nil
    FloatingBtnGui = nil
    statsGuiRef = nil

    pcall(function() LogService:ClearOutput() end)
    print("[Private Script] Encerrado pelo usuário. Execute novamente para reativar.")
end

local function openLauncher()
    if not Utils.alive() then return end
    local T = UI.Theme
    local isMobile = Utils.isMobile()
    local FULL = isMobile and Utils.scaledSize(400, 210) or Utils.scaledSize(560, 360)

    local gui = Utils.create("ScreenGui", {
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true, Name = "LauncherGui",
    }, PARENT_GUI)

    local main = Utils.create("Frame", {
        AnchorPoint = Vector2.new(0.5,0.5),
        BackgroundColor3 = T.Container,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Position = UDim2.new(0.5,0,0.5,0),
        Size = UDim2.new(0,0,0,0),
        ClipsDescendants = true,
    }, gui)
    UI.corner(main, UDim.new(0, 14))
    local mainStroke = Utils.create("UIStroke", {
        Color = T.Accent, Thickness = 1.5, Transparency = 1,
    }, main)

    Utils.addUiBackground(main, 0.75)

    local topBar = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 16, 0, 32),
        Size = UDim2.new(1, -32, 0, 44),
        ZIndex = 2,
    }, main)
    Utils.makeDraggable(topBar, main)

    local titleLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack,
        Text = "PRIVATE SCRIPT",
        TextColor3 = T.Text,
        TextTransparency = 1,
        TextSize = isMobile and 15 or 19,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, -40, 0, 22),
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 3,
    }, topBar)

    local welcomeLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium,
        Text = CFG.TITLE_WELCOME,
        TextColor3 = T.Welcome,
        TextTransparency = 1,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 0, 0, isMobile and 20 or 22),
        Size = UDim2.new(1, -40, 0, 16),
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 3,
    }, topBar)

    local closeBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "✕",
        TextColor3 = T.TextMuted, TextTransparency = 1,
        TextSize = 14,
        BackgroundColor3 = T.CardDark,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        AnchorPoint = Vector2.new(1,0),
        Position = UDim2.new(1,0,0,2),
        Size = UDim2.new(0,28,0,28),
        AutoButtonColor = false,
        ZIndex = 5,
    }, topBar)
    UI.corner(closeBtn, UDim.new(1, 0))
    local closeStroke = Utils.create("UIStroke",
        {Color = T.Border, Thickness = 1, Transparency = 1}, closeBtn)

    closeBtn.MouseEnter:Connect(function()
        TweenService:Create(closeBtn, TweenInfo.new(0.15),
            {BackgroundColor3 = T.Danger, TextColor3 = Color3.fromRGB(255,255,255)}):Play()
    end)
    closeBtn.MouseLeave:Connect(function()
        TweenService:Create(closeBtn, TweenInfo.new(0.15),
            {BackgroundColor3 = T.CardDark, TextColor3 = T.TextMuted}):Play()
    end)

    local content = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,0,0,isMobile and 66 or 72),
        Size = UDim2.new(1,0,1,-(isMobile and 66 or 72)),
        ZIndex = 2,
    }, main)

    local function dismiss(cb)
        TweenService:Create(main, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Size = UDim2.new(0, FULL.X.Offset * 0.6, 0, FULL.Y.Offset * 0.6),
            BackgroundTransparency = 1,
        }):Play()
        TweenService:Create(mainStroke, TweenInfo.new(0.22), {Transparency = 1}):Play()
        pcall(function()
            for _, d in ipairs(main:GetDescendants()) do
                if d:IsA("TextLabel") or d:IsA("TextButton") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.18), {TextTransparency = 1}):Play()
                    end)
                elseif d:IsA("ImageLabel") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.18), {ImageTransparency = 1}):Play()
                    end)
                elseif d:IsA("UIStroke") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.18), {Transparency = 1}):Play()
                    end)
                elseif d:IsA("Frame") and d ~= main then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.18), {BackgroundTransparency = 1}):Play()
                    end)
                end
            end
        end)
        task.wait(0.24)
        gui:Destroy()
        if cb then cb() end
    end

    closeBtn.Activated:Connect(function()
        if not Utils.alive() then return end
        TweenService:Create(main, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Size = UDim2.new(0, FULL.X.Offset * 0.4, 0, FULL.Y.Offset * 0.4),
            BackgroundTransparency = 1,
        }):Play()
        pcall(function()
            for _, d in ipairs(main:GetDescendants()) do
                if d:IsA("TextLabel") or d:IsA("TextButton") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.15), {TextTransparency = 1}):Play()
                    end)
                elseif d:IsA("ImageLabel") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.15), {ImageTransparency = 1}):Play()
                    end)
                elseif d:IsA("UIStroke") then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.15), {Transparency = 1}):Play()
                    end)
                elseif d:IsA("Frame") and d ~= main then
                    pcall(function()
                        TweenService:Create(d, TweenInfo.new(0.15), {BackgroundTransparency = 1}):Play()
                    end)
                end
            end
        end)
        task.wait(0.22)
        shutdownScript()
    end)

    local function makeBigButton(parent, xOffset, colorAccent, iconId, titleText, subText, onClick, delay)
        local BTN_W = isMobile and 168 or 210
        local BTN_H = isMobile and 118 or 178

        local btn = Utils.create("TextButton", {
            BackgroundColor3 = T.Card,
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Position = UDim2.new(0, xOffset, 0, 30),
            Size = UDim2.new(0, BTN_W, 0, BTN_H),
            Text = "", AutoButtonColor = false,
            ZIndex = 3,
        }, parent)
        UI.corner(btn, UDim.new(0, 14))
        Utils.addBtnBg(btn, 0.15)

        local stroke = Utils.create("UIStroke", {
            Color = colorAccent, Thickness = 1.5, Transparency = 1,
        }, btn)

        local glow = Utils.create("Frame", {
            BackgroundColor3 = colorAccent,
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Position = UDim2.new(0, 0, 0, 0),
            Size = UDim2.new(1, 0, 0, 3),
            ZIndex = 3,
        }, btn)
        UI.corner(glow, UDim.new(1, 0))

        local icon = Utils.create("ImageLabel", {
            BackgroundTransparency = 1,
            Image = iconId,
            ImageColor3 = colorAccent,
            ImageTransparency = 1,
            ScaleType = Enum.ScaleType.Fit,
            AnchorPoint = Vector2.new(0.5, 0),
            Position = UDim2.new(0.5, 0, 0, isMobile and 14 or 20),
            Size = UDim2.new(0, isMobile and 44 or 64, 0, isMobile and 44 or 64),
            ZIndex = 4,
        }, btn)

        local titleLbl = Utils.create("TextLabel", {
            Font = Enum.Font.GothamBlack, Text = titleText,
            TextColor3 = colorAccent,
            TextTransparency = 1,
            TextSize = isMobile and 22 or 30,
            BackgroundTransparency = 1,
            Position = UDim2.new(0,0,0, isMobile and 62 or 92),
            Size = UDim2.new(1,0,0, isMobile and 26 or 34),
            TextXAlignment = Enum.TextXAlignment.Center,
            ZIndex = 4,
        }, btn)

        local subLbl = Utils.create("TextLabel", {
            Font = Enum.Font.GothamMedium, Text = subText,
            TextColor3 = T.TextMuted,
            TextTransparency = 1,
            TextSize = isMobile and 9 or 10,
            BackgroundTransparency = 1,
            Position = UDim2.new(0,0,0, isMobile and 88 or 130),
            Size = UDim2.new(1,0,0, isMobile and 24 or 40),
            TextXAlignment = Enum.TextXAlignment.Center,
            TextWrapped = true,
            ZIndex = 4,
        }, btn)

        task.delay(delay or 0, function()
            if not (btn and btn.Parent) then return end
            TweenService:Create(btn, TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                Position = UDim2.new(0, xOffset, 0, 10),
                BackgroundTransparency = 0,
            }):Play()
            TweenService:Create(stroke, TweenInfo.new(0.4), {Transparency = 0.3}):Play()
            TweenService:Create(glow, TweenInfo.new(0.4), {BackgroundTransparency = 0.6}):Play()
            TweenService:Create(icon, TweenInfo.new(0.35), {ImageTransparency = 0}):Play()
            TweenService:Create(titleLbl, TweenInfo.new(0.35), {TextTransparency = 0}):Play()
            task.delay(0.12, function()
                if subLbl and subLbl.Parent then
                    TweenService:Create(subLbl, TweenInfo.new(0.35), {TextTransparency = 0}):Play()
                end
            end)
        end)

        local origSize = btn.Size
        btn.MouseEnter:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.2),
                {Size = UDim2.new(0, BTN_W + 6, 0, BTN_H + 6),
                 Position = UDim2.new(0, xOffset - 3, 0, 7)}):Play()
            TweenService:Create(stroke, TweenInfo.new(0.2), {Transparency = 0, Thickness = 2}):Play()
        end)
        btn.MouseLeave:Connect(function()
            TweenService:Create(btn, TweenInfo.new(0.2),
                {Size = origSize,
                 Position = UDim2.new(0, xOffset, 0, 10)}):Play()
            TweenService:Create(stroke, TweenInfo.new(0.2), {Transparency = 0.3, Thickness = 1.5}):Play()
        end)

        btn.Activated:Connect(function()
            if not Utils.alive() then return end
            local pulse = TweenService:Create(btn,
                TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                {Size = UDim2.new(0, BTN_W - 6, 0, BTN_H - 6),
                 Position = UDim2.new(0, xOffset + 3, 0, 13)})
            pulse:Play()
            pulse.Completed:Wait()
            TweenService:Create(btn, TweenInfo.new(0.15, Enum.EasingStyle.Back),
                {Size = UDim2.new(0, BTN_W + 4, 0, BTN_H + 4),
                 Position = UDim2.new(0, xOffset - 2, 0, 8)}):Play()
            task.wait(0.15)
            dismiss(function() onClick() end)
        end)

        return btn
    end

    local BTN_W = isMobile and 168 or 210
    local GAP = 16
    local totalW = BTN_W * 2 + GAP
    local leftOffset = (FULL.X.Offset - totalW) / 2

    makeBigButton(
        content,
        leftOffset,
        T.Fast,
        CFG.LAUNCHER_ICON_ID,
        "FAST",
        "Auto Lift + Maquinas",
        function() openAutoFastWindow() end,
        0.35
    )

    makeBigButton(
        content,
        leftOffset + BTN_W + GAP,
        T.Success,
        CFG.LAUNCHER_ICON_ID,
        "MENU",
        "Farm + Tools\nStats + TP + Boss",
        function() openMainHub() end,
        0.5
    )

    local versionLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium,
        Text = "v18.12.15  •  Kil09122",
        TextColor3 = T.TextDark,
        TextTransparency = 1,
        TextSize = 10,
        BackgroundTransparency = 1,
        AnchorPoint = Vector2.new(0.5, 1),
        Position = UDim2.new(0.5, 0, 1, -8),
        Size = UDim2.new(1, 0, 0, 14),
        TextXAlignment = Enum.TextXAlignment.Center,
        ZIndex = 3,
    }, main)
    task.delay(0.6, function()
        if versionLbl and versionLbl.Parent then
            TweenService:Create(versionLbl, TweenInfo.new(0.4), {TextTransparency = 0}):Play()
        end
    end)

    task.delay(0.05, function()
        if not (main and main.Parent) then return end
        TweenService:Create(main,
            TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size = FULL, BackgroundTransparency = 0}):Play()
        TweenService:Create(mainStroke,
            TweenInfo.new(0.4), {Transparency = 0.2}):Play()

        task.delay(0.15, function()
            if not (topBar and topBar.Parent) then return end
            TweenService:Create(topBar,
                TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                {Position = UDim2.new(0, 16, 0, 12)}):Play()
            TweenService:Create(titleLbl, TweenInfo.new(0.35), {TextTransparency = 0}):Play()
            TweenService:Create(welcomeLbl, TweenInfo.new(0.35), {TextTransparency = 0}):Play()
            TweenService:Create(closeBtn, TweenInfo.new(0.35), {TextTransparency = 0, BackgroundTransparency = 0}):Play()
            TweenService:Create(closeStroke, TweenInfo.new(0.35), {Transparency = 0.4}):Play()
        end)
    end)
end

--==========================================================
-- 17. FAST WINDOW
--==========================================================
function openAutoFastWindow()
    if not Utils.alive() then return end
    if AutoFastGui then AutoFastGui:Destroy() end
    State.lastWindow = "fast"
    local T = UI.Theme
    local isMobile = Utils.isMobile()

    local FULL = isMobile and Utils.scaledSize(450, 450) or Utils.scaledSize(620, 560)

    AutoFastGui = Utils.create("ScreenGui", {
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true, Name = "AutoFastGui",
    }, PARENT_GUI)

    local main = Utils.create("Frame", {
        AnchorPoint = Vector2.new(0.5,0.5), BackgroundColor3 = T.Container,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Position = UDim2.new(0.5,0,0.5,0),
        Size = UDim2.new(0,0,0,0),
        ClipsDescendants = true,
    }, AutoFastGui)
    UI.corner(main, UDim.new(0, 14))
    local mainStroke = Utils.create("UIStroke", {Color = T.Accent, Thickness = 1.5, Transparency = 1}, main)

    Utils.addUiBackground(main, 0.75)

    local topBar = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,16,0,10),
        Size = UDim2.new(1,-32,0,40),
    }, main)
    Utils.makeDraggable(topBar, main)

    local titleFrame = Utils.create("Frame", {
        BackgroundTransparency = 1, Position = UDim2.new(0, 0, 0, 0),
        Size = UDim2.new(1, -70, 1, 0),
    }, topBar)

    local dot = Utils.create("Frame", {
        BackgroundColor3 = T.Fast, BorderSizePixel = 0,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 0, 0.5, 0),
        Size = UDim2.new(0, 8, 0, 8),
    }, titleFrame)
    UI.corner(dot, UDim.new(1, 0))

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "FAST FARMING",
        TextColor3 = T.Fast, TextSize = isMobile and 12 or 16,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 16, 0, 0),
        Size = UDim2.new(0, isMobile and 100 or 130, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
    }, titleFrame)

    Utils.create("Frame", {
        BackgroundColor3 = T.Border, BorderSizePixel = 0,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, isMobile and 118 or 148, 0.5, 0),
        Size = UDim2.new(0, 1, 0, 14),
    }, titleFrame)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = CFG.TITLE_WELCOME,
        TextColor3 = T.Welcome, TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, isMobile and 128 or 158, 0, 0),
        Size = UDim2.new(1, isMobile and -128 or -158, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
    }, titleFrame)

    local contentContainer = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,0,0,58),
        Size = UDim2.new(1,0,1,-58),
    }, main)

    UI.createWindowButtons(topBar,
        function()
            TweenService:Create(main, TweenInfo.new(0.25), {Size = UDim2.new(0,0,0,0)}):Play()
            task.wait(0.3)
            AutoFastGui:Destroy(); AutoFastGui = nil
            createFloatingButton()
        end,
        function()
            TweenService:Create(main, TweenInfo.new(0.25), {Size = UDim2.new(0,0,0,0)}):Play()
            task.wait(0.3)
            AutoFastGui:Destroy(); AutoFastGui = nil
            if not State.antiAfk.active then
                if statsGuiRef and statsGuiRef.Parent then
                    pcall(function() statsGuiRef:Destroy() end)
                end
                statsGuiRef = nil
            end
            createFloatingButton()
        end,
        function() contentContainer.Visible = true
            TweenService:Create(main, TweenInfo.new(0.25), {Size = FULL}):Play() end
    )

    local tabsArea = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark,
        BorderSizePixel = 0,
        Position = UDim2.new(0,16,0,0),
        Size = UDim2.new(1,-32,0,36),
        ClipsDescendants = true,
    }, contentContainer)
    UI.corner(tabsArea, UDim.new(0, 10))

    local TAB_W = isMobile and 80 or 100
    local TAB_GAP = 12
    local tabInfo = {}
    local tabOrder = {}

    local tabsContainer = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,4,0,4),
        Size = UDim2.new(0, 9999, 1, -8),
    }, tabsArea)

    local tabIndicator = Utils.create("Frame", {
        BackgroundColor3 = T.Accent,
        BorderSizePixel = 0,
        Position = UDim2.new(0, 4, 1, -3),
        Size = UDim2.new(0, TAB_W, 0, 2),
        ZIndex = 6,
    }, tabsArea)
    UI.corner(tabIndicator, UDim.new(1, 0))

    local function makeTab(i, name)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = name, TextColor3 = T.TextDark,
            TextSize = isMobile and 9 or 11,
            BackgroundColor3 = Color3.fromRGB(0,0,0),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Position = UDim2.new(0, (i-1)*(TAB_W+TAB_GAP), 0, 0),
            Size = UDim2.new(0, TAB_W, 1, 0),
            AutoButtonColor = false,
        }, tabsContainer)
        UI.corner(b, UDim.new(0, 6))
        tabInfo[name] = { btn = b }
        table.insert(tabOrder, name)

        if i > 1 then
            local sep = Utils.create("Frame", {
                BackgroundColor3 = Color3.fromRGB(255, 255, 255),
                BackgroundTransparency = 1,
                BorderSizePixel = 0,
                AnchorPoint = Vector2.new(1, 0.5),
                Position = UDim2.new(0, -6, 0.5, 0),
                Size = UDim2.new(0, 2, 0, 14),
                ZIndex = 6,
            }, b)

            local sepImg = Utils.create("ImageLabel", {
                BackgroundTransparency = 1,
                Image = CFG.BTN_BG_IMAGE,
                ImageTransparency = 0.3,
                ImageColor3 = T.Border,
                ScaleType = Enum.ScaleType.Stretch,
                Size = UDim2.new(1, 0, 1, 0),
                ZIndex = 6,
            }, sep)

            local gradient = Instance.new("UIGradient")
            gradient.Rotation = 90
            gradient.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, 1),
                NumberSequenceKeypoint.new(0.2, 0),
                NumberSequenceKeypoint.new(0.8, 0),
                NumberSequenceKeypoint.new(1, 1)
            })
            gradient.Parent = sepImg
            Utils.create("UICorner", {CornerRadius = UDim.new(1, 0)}, sepImg)
        end

        return b
    end

    local FAST_TABS = {"Main", "Fast Farming", "Rebithers", "Config"}
    for i, name in ipairs(FAST_TABS) do makeTab(i, name) end

    local contentArea = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark,
        BorderSizePixel = 0,
        Position = UDim2.new(0,16,0,58),
        Size = UDim2.new(1,-32,1,-74),
        ClipsDescendants = true,
        ZIndex = 1,
    }, contentContainer)
    UI.corner(contentArea, UDim.new(0, 10))
    Utils.create("UIStroke",
        {Color = T.Border, Thickness = 1, Transparency = 0.4}, contentArea)

    local accentLabels = {}
    local function makeScroll()
        local s = Utils.create("ScrollingFrame", {
            BackgroundTransparency = 1,
            Position = UDim2.new(0,10,0,10),
            Size = UDim2.new(1,-20,1,-20),
            CanvasSize = UDim2.new(0,0,0,2000),
            ScrollBarThickness = 4,
            ScrollBarImageColor3 = T.Accent,
            ScrollBarImageTransparency = 0.3,
            Visible = false,
            ZIndex = 5,
        }, contentArea)
        Utils.create("UIListLayout",
            {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 14)}, s)
        Utils.create("UIPadding",
            {PaddingTop = UDim.new(0,4), PaddingBottom = UDim.new(0,20),
             PaddingLeft = UDim.new(0,4), PaddingRight = UDim.new(0,4)}, s)
        return s
    end

    local CARD_H = isMobile and 44 or 64

    local function makeCard(parent, order, height)
        local wrap = Utils.create("Frame", {
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Size = UDim2.new(1, 0, 0, (height or CARD_H)),
            LayoutOrder = order,
            ZIndex = 5,
        }, parent)
        
        local card = Utils.create("Frame", {
            BackgroundColor3 = T.Card,
            BackgroundTransparency = 0.35,
            BorderSizePixel = 0,
            Size = UDim2.new(1,0,0, height or CARD_H),
            LayoutOrder = 0,
            ZIndex = 5,
        }, wrap)
        UI.corner(card, UDim.new(0, 10))
        Utils.addBtnBg(card, 0.15)
        Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, card)
        return card
    end

    -- ---------- MAIN ----------
    local mainScroll = makeScroll()
    mainScroll.Visible = true

    local devCard = makeCard(mainScroll, 1)
    local devTitle = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "DEVELOPER INFO", TextColor3 = T.Accent,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,6),
        Size = UDim2.new(1,-28,0,14), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, devCard)
    table.insert(accentLabels, devTitle)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "Criador / DEVELOP: Kil09122", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13, BackgroundTransparency = 1,
        Position = UDim2.new(0,14,0,22),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, devCard)

    local antiCard = makeCard(mainScroll, 3)
    local antiStroke = Utils.create("UIStroke",
        {Color = State.antiAfk.active and T.Success or T.Border,
         Thickness = State.antiAfk.active and 2 or 1,
         Transparency = State.antiAfk.active and 0.1 or 0.4}, antiCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "ANTI AFK", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, antiCard)
    local antiBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = State.antiAfk.active and "ON" or "OFF",
        TextColor3 = State.antiAfk.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = State.antiAfk.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 76 or 100, 0, isMobile and 28 or 36),
        AutoButtonColor = false, ZIndex = 6,
    }, antiCard)
    UI.corner(antiBtn, UDim.new(1,0))
    Utils.create("UIStroke", {
        Color = State.antiAfk.active and T.Success or T.Border,
        Thickness = 1, Transparency = 0.3,
    }, antiBtn)

    local function buildFastStatsGui()
        if statsGuiRef and statsGuiRef.Parent then
            pcall(function() statsGuiRef:Destroy() end)
        end
        statsGuiRef = Utils.create("ScreenGui", {
            ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
            IgnoreGuiInset = true, Name = "PS_StatsGui",
            Enabled = State.antiAfk.active,
        }, PARENT_GUI)
        local statsGui = statsGuiRef

        local statsBox = Utils.create("Frame", {
            BackgroundColor3 = T.Container, BorderSizePixel = 0,
            Position = UDim2.new(0, 40, 0, 40), Size = UDim2.new(0, 260, 0, 172),
            ClipsDescendants = true,
        }, statsGui)
        UI.corner(statsBox, UDim.new(0, 12))
        Utils.addBtnBg(statsBox, 0.3)
        Utils.create("UIStroke", {Color = T.Accent, Thickness = 1.5, Transparency = 0.3}, statsBox)
        Utils.makeDraggable(statsBox, statsBox)

        statsBox.Size = UDim2.new(0, 0, 0, 0)
        TweenService:Create(statsBox, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
            Size = UDim2.new(0, 260, 0, 172)
        }):Play()

        local headerRow = Utils.create("Frame", {
            BackgroundTransparency = 1,
            Position = UDim2.new(0, 12, 0, 10),
            Size = UDim2.new(1, -24, 0, 22),
            ZIndex = 6,
        }, statsBox)

        local pumpkin = Utils.create("ImageLabel", {
            BackgroundTransparency = 1,
            Image = CFG.BANNER_IMAGE,
            ScaleType = Enum.ScaleType.Crop,
            AnchorPoint = Vector2.new(0, 0.5),
            Position = UDim2.new(0, 0, 0.5, 0),
            Size = UDim2.new(0, 20, 0, 20),
            ZIndex = 6,
        }, headerRow)
        UI.corner(pumpkin, UDim.new(1, 0))
        Utils.create("UIStroke", {
            Color = T.Accent, Thickness = 1.5, Transparency = 0.2,
        }, pumpkin)

        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = "ANTI AFK - STATUS", TextColor3 = T.Accent,
            TextSize = 12, BackgroundTransparency = 1,
            Position = UDim2.new(0, 26, 0, 0),
            Size = UDim2.new(1, -50, 1, 0),
            TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, headerRow)

        local minAntiBtn = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = "—",
            TextColor3 = T.TextMuted, TextSize = 14,
            BackgroundColor3 = T.CardDark,
            BorderSizePixel = 0,
            AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.new(1, 0, 0.5, 0),
            Size = UDim2.new(0, 20, 0, 20),
            AutoButtonColor = false,
            ZIndex = 7,
        }, headerRow)
        UI.corner(minAntiBtn, UDim.new(1,0))
        minAntiBtn.Activated:Connect(function()
            TweenService:Create(statsBox, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
                Size = UDim2.new(0, 0, 0, 0)
            }):Play()
            task.wait(0.3)
            if statsGuiRef and statsGuiRef.Parent then
                statsGuiRef.Enabled = false
            end
            State.antiAfk.isOpen = false
        end)

        Utils.create("Frame", {
            BackgroundColor3 = T.BorderSoft, BorderSizePixel = 0,
            Position = UDim2.new(0,14,0,38), Size = UDim2.new(1,-28,0,1),
            ZIndex = 6,
        }, statsBox)
        local statImg = Utils.create("ImageLabel", {
            BackgroundColor3 = T.Card, BorderSizePixel = 0,
            Position = UDim2.new(0,14,0,50), Size = UDim2.new(0, 56, 0, 56),
            Image = Utils.getPlayerImage(), ScaleType = Enum.ScaleType.Crop,
            ZIndex = 6,
        }, statsBox)
        UI.corner(statImg, UDim.new(0, 10))
        Utils.create("UIStroke", {Color = T.Success, Thickness = 2, Transparency = 0.4}, statImg)

        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = Player.Name,
            TextColor3 = T.Text, TextSize = 12, BackgroundTransparency = 1,
            Position = UDim2.new(0,82,0,50), Size = UDim2.new(1,-96,0,20),
            TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
        }, statsBox)

        local fpsLbl = Utils.create("TextLabel", {
            Font = Enum.Font.GothamMedium, Text = "FPS: --", TextColor3 = T.Success,
            TextSize = 11, BackgroundTransparency = 1, Position = UDim2.new(0,82,0,70),
            Size = UDim2.new(1,-96,0,18), TextXAlignment = Enum.TextXAlignment.Left,
            ZIndex = 6,
        }, statsBox)
        local pingLbl = Utils.create("TextLabel", {
            Font = Enum.Font.GothamMedium, Text = "Ping: -- ms",
            TextColor3 = T.Gold, TextSize = 11,
            BackgroundTransparency = 1, Position = UDim2.new(0,82,0,88),
            Size = UDim2.new(1,-96,0,18), TextXAlignment = Enum.TextXAlignment.Left,
            ZIndex = 6,
        }, statsBox)
        local uptimeLbl = Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = "Tempo: 00:00:00",
            TextColor3 = T.Welcome, TextSize = 12,
            BackgroundTransparency = 1, Position = UDim2.new(0,14,0,120),
            Size = UDim2.new(1,-28,0,22), TextXAlignment = Enum.TextXAlignment.Left,
            ZIndex = 6,
        }, statsBox)

        task.spawn(function()
            while Utils.alive() and statsGui and statsGui.Parent do
                pcall(function()
                    if State.antiAfk.active then
                        local fps = math.floor(60 * (1 / math.max(workspace:GetRealPhysicsFPS(), 1)))
                        fpsLbl.Text = "FPS: " .. tostring(fps)
                        local pv = 0
                        pcall(function() pv = math.round(Player:GetNetworkPing() * 1000) end)
                        pingLbl.Text = "Ping: " .. tostring(pv) .. " ms"
                        local elapsed = math.floor(tick() - State.antiAfk.startTime + State.antiAfk.totalTime)
                        uptimeLbl.Text = string.format("Tempo: %02d:%02d:%02d",
                            math.floor(elapsed/3600), math.floor((elapsed%3600)/60), elapsed%60)
                    end
                end)
                task.wait(1)
            end
        end)
    end

    antiBtn.Activated:Connect(function()
        State.antiAfk.active = not State.antiAfk.active
        if State.antiAfk.active then
            State.antiAfk.startTime = tick()
            State.antiAfk.totalTime = 0
            antiBtn.Text = "ON"; antiBtn.TextColor3 = T.Text
            antiBtn.BackgroundColor3 = T.Success
            antiStroke.Color = T.Success; antiStroke.Transparency = 0.1
            
            if not (statsGuiRef and statsGuiRef.Parent) then
                State.antiAfk.isOpen = true
                buildFastStatsGui()
            else
                State.antiAfk.isOpen = true
                statsGuiRef.Enabled = true
                local box = statsGuiRef:FindFirstChildOfClass("Frame")
                if box then
                    box.Size = UDim2.new(0, 0, 0, 0)
                    TweenService:Create(box, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                        Size = UDim2.new(0, 260, 0, 172)
                    }):Play()
                end
            end
        else
            State.antiAfk.totalTime = 0
            State.antiAfk.startTime = 0
            antiBtn.Text = "OFF"; antiBtn.TextColor3 = T.TextMuted
            antiBtn.BackgroundColor3 = T.CardDark
            antiStroke.Color = T.Border; antiStroke.Transparency = 0.4
            if statsGuiRef and statsGuiRef.Parent then
                statsGuiRef.Enabled = false
            end
            State.antiAfk.isOpen = false
        end
    end)

    if State.antiAfk.active and not (statsGuiRef and statsGuiRef.Parent) then
        buildFastStatsGui()
    end

    task.spawn(function()
        while AutoFastGui and AutoFastGui.Parent and Utils.alive() do
            pcall(function()
                if antiBtn and antiBtn.Parent then
                    local ativo = State.antiAfk.active
                    if ativo then
                        antiBtn.Text = "ON"; antiBtn.TextColor3 = T.Text
                        antiBtn.BackgroundColor3 = T.Success
                        antiStroke.Color = T.Success; antiStroke.Transparency = 0.1
                    else
                        antiBtn.Text = "OFF"; antiBtn.TextColor3 = T.TextMuted
                        antiBtn.BackgroundColor3 = T.CardDark
                        antiStroke.Color = T.Border; antiStroke.Transparency = 0.4
                    end
                end
            end)
            task.wait(0.3)
        end
    end)

    local updCard = makeCard(mainScroll, 4)
    local updTitle = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "ATUALIZAÇÕES", TextColor3 = T.Accent,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,6),
        Size = UDim2.new(1,-28,0,14), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, updCard)
    table.insert(accentLabels, updTitle)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium,
        Text = "v18.12.15 - Detector de Força Moderno",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12, TextWrapped = true,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,22),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Top, ZIndex = 6,
    }, updCard)

    local discordBtn = Utils.create("TextButton", {
        BackgroundColor3 = T.Discord, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 38 or 46), LayoutOrder = 5,
        AutoButtonColor = false, Text = "", ZIndex = 5,
    }, mainScroll)
    UI.corner(discordBtn, UDim.new(0, 10))
    Utils.addBtnBg(discordBtn, 0.5)
    Utils.create("UIStroke", {Color = T.DiscordHi, Thickness = 1, Transparency = 0.5}, discordBtn)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Copiar Link do Discord", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13, BackgroundTransparency = 1,
        Position = UDim2.new(0,44,0,0),
        Size = UDim2.new(1,-52,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, discordBtn)

    local discordIcon = Utils.create("ImageLabel", {
        BackgroundTransparency = 1,
        Image = CFG.DISCORD_ICON,
        ScaleType = Enum.ScaleType.Crop,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 12, 0.5, 0),
        Size = UDim2.new(0, 22, 0, 22),
        ZIndex = 6,
    }, discordBtn)
    UI.corner(discordIcon, UDim.new(1, 0))

    discordBtn.MouseEnter:Connect(function()
        TweenService:Create(discordBtn, TweenInfo.new(0.2), {BackgroundColor3 = T.DiscordHi}):Play()
    end)
    discordBtn.MouseLeave:Connect(function()
        TweenService:Create(discordBtn, TweenInfo.new(0.2), {BackgroundColor3 = T.Discord}):Play()
    end)
    discordBtn.Activated:Connect(function()
        if setclipboard then setclipboard("https://discord.gg/WRphBR4AV") end
    end)

    -- ---------- FAST FARMING ----------
    local farmScroll = makeScroll()

    local speedCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0, isMobile and 160 or 190),
        LayoutOrder = 1, ZIndex = 5,
    }, farmScroll)
    UI.corner(speedCard, UDim.new(0, 10))
    Utils.addBtnBg(speedCard, 0.15)
    Utils.create("UIStroke", {Color = T.Fast, Thickness = 1.5, Transparency = 0.4}, speedCard)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "AUTO FAST",
        TextColor3 = T.Fast, TextSize = isMobile and 16 or 20,
        BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,16,0,26),
        Size = UDim2.new(1, isMobile and -110 or -180, 0, 30),
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, speedCard)

    local toggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = AutoFast.active and "ON" or "OFF",
        TextColor3 = AutoFast.active and T.Text or T.TextMuted,
        TextSize = isMobile and 14 or 16,
        BackgroundColor3 = AutoFast.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-16,0,26),
        Size = UDim2.new(0, isMobile and 80 or 110, 0, isMobile and 34 or 42),
        AutoButtonColor = false, ZIndex = 6,
    }, speedCard)
    UI.corner(toggle, UDim.new(1,0))

    local cardStroke = Utils.create("UIStroke", {Color = T.Fast, Thickness = 1.5, Transparency = AutoFast.active and 0.1 or 0.4}, speedCard)

    toggle.Activated:Connect(function()
        AutoFast.active = not AutoFast.active
        if AutoFast.active then
            toggle.Text = "ON"; toggle.TextColor3 = T.Text
            toggle.BackgroundColor3 = T.Success
            cardStroke.Transparency = 0.1
            AutoFast.findMuscleEvent()
            AutoFast.findToolRemote()
        else
            AutoFast.stopRepLoop()
            toggle.Text = "OFF"; toggle.TextColor3 = T.TextMuted
            toggle.BackgroundColor3 = T.CardDark
            cardStroke.Transparency = 0.4
        end
    end)

    local statusLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "Aguardando...",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,16,0,56),
        Size = UDim2.new(1,-32,0,16), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, speedCard)

    task.spawn(function()
        while AutoFastGui and AutoFastGui.Parent and Utils.alive() do
            pcall(function()
                if AutoFast.mode == "rep" then
                    statusLbl.Text = "REP: " .. (AutoFast.target or "...")
                    statusLbl.TextColor3 = T.Welcome
                else
                    statusLbl.Text = "Desativado"
                    statusLbl.TextColor3 = T.TextDark
                end
            end)
            task.wait(0.5)
        end
    end)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "REPS POR CICLO (a cada 0.01s)", TextColor3 = T.Accent,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,16,0,88),
        Size = UDim2.new(1,-32,0,16), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, speedCard)

    local SPEEDS = { ["10"]=10, ["15"]=15, ["30"]=30, ["60"]=60, ["120"]=120 }
    local speedKeys = {"10","15","30","60","120"}
    local speedBtns = {}

    CFG.AUTO_FAST_DELAY = 0.01
    local currentSpeedKey = tostring(CFG.REP_BATCH or 10)
    if not SPEEDS[currentSpeedKey] then
        currentSpeedKey = "10"
        CFG.REP_BATCH = 10
    end

    local function refreshSpeed()
        for k, b in pairs(speedBtns) do
            if k == currentSpeedKey then
                b.BackgroundColor3 = T.Success
                b.TextColor3 = T.Text
            else
                b.BackgroundColor3 = T.CardDark
                b.TextColor3 = T.TextMuted
            end
        end
    end

    local BTN_W = isMobile and 62 or 82
    local BTN_GAP = isMobile and 4 or 5

    for i, key in ipairs(speedKeys) do
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = key,
            TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12,
            BackgroundColor3 = T.CardDark,
            BorderSizePixel = 0, AutoButtonColor = false,
            Position = UDim2.new(0, 16 + (i-1)*(BTN_W+BTN_GAP), 0, isMobile and 112 or 118),
            Size = UDim2.new(0, BTN_W, 0, isMobile and 28 or 34),
            ZIndex = 6,
        }, speedCard)
        UI.corner(b, UDim.new(0, 6))
        speedBtns[key] = b
        b.Activated:Connect(function()
            currentSpeedKey = key
            CFG.REP_BATCH = SPEEDS[key]
            CFG.AUTO_FAST_DELAY = 0.01
            refreshSpeed()
        end)
    end
    refreshSpeed()

    -- ---------- REBITHERS ----------
    local rebirthScroll = makeScroll()
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Rebirths", TextColor3 = T.Rebirth,
        TextSize = isMobile and 12 or 15,
        BackgroundTransparency = 1, Size = UDim2.new(1,0,0,20),
        LayoutOrder = 1, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, rebirthScroll)

    local function makeToggleCard(parent, order, labelText)
        local card = makeCard(parent, order)
        local stroke = Utils.create("UIStroke",
            {Color = T.Border, Thickness = 1, Transparency = 0.4}, card)
        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = labelText, TextColor3 = T.Text,
            TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
            Position = UDim2.new(0,16,0.5,0),
            Size = UDim2.new(1,-140,1,0), TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, card)
        local btn = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = "OFF",
            TextColor3 = T.TextMuted, TextSize = isMobile and 11 or 13,
            BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
            AnchorPoint = Vector2.new(1,0.5),
            Position = UDim2.new(1,-16,0.5,0),
            Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
            AutoButtonColor = false, ZIndex = 6,
        }, card)
        UI.corner(btn, UDim.new(1,0))
        return card, btn, stroke
    end

    local rebirthCard, rebirthToggle, rebirthStroke = makeToggleCard(rebirthScroll, 2, "Auto Rebirth")
    rebirthToggle.Activated:Connect(function()
        AutoRebirth.active = not AutoRebirth.active
        if AutoRebirth.active then
            AutoRebirth.lastFire = 0
            rebirthToggle.Text = "ON"; rebirthToggle.TextColor3 = T.Text
            rebirthToggle.BackgroundColor3 = T.Success
            rebirthStroke.Color = T.Success; rebirthStroke.Transparency = 0.1
        else
            rebirthToggle.Text = "OFF"; rebirthToggle.TextColor3 = T.TextMuted
            rebirthToggle.BackgroundColor3 = T.CardDark
            rebirthStroke.Color = T.Border; rebirthStroke.Transparency = 0.4
        end
    end)

    local tpCard, tpToggle, tpStroke = makeToggleCard(rebirthScroll, 3, "Auto TP Muscle King")
    tpToggle.Activated:Connect(function()
        State.autoTP.active = not State.autoTP.active
        AutoTP.active = State.autoTP.active
        if State.autoTP.active then
            AutoTP.lastTP = 0
            tpToggle.Text = "ON"; tpToggle.TextColor3 = T.Text
            tpToggle.BackgroundColor3 = T.Success
            tpStroke.Color = T.Success; tpStroke.Transparency = 0.1
            State.autoTP.keepLock = true
            State.lockPos.active = true
            State.lockPos.pos = AutoTP.customPos
        else
            tpToggle.Text = "OFF"; tpToggle.TextColor3 = T.TextMuted
            tpToggle.BackgroundColor3 = T.CardDark
            tpStroke.Color = T.Border; tpStroke.Transparency = 0.4
            State.autoTP.keepLock = false
            State.lockPos.active = false
        end
    end)

    local lockCard, lockToggle, lockStroke = makeToggleCard(rebirthScroll, 4, "Lock Position")
    lockToggle.Activated:Connect(function()
        State.lockPos.active = not State.lockPos.active
        if State.lockPos.active then
            local char = Player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            State.lockPos.pos = hrp and hrp.Position or nil
            lockToggle.Text = "ON"; lockToggle.TextColor3 = T.Text
            lockToggle.BackgroundColor3 = T.Success
            lockStroke.Color = T.Success; lockStroke.Transparency = 0.1
        else
            State.lockPos.pos = nil
            lockToggle.Text = "OFF"; lockToggle.TextColor3 = T.TextMuted
            lockToggle.BackgroundColor3 = T.CardDark
            lockStroke.Color = T.Border; lockStroke.Transparency = 0.4
        end
    end)

    local sizeCard, sizeToggle, sizeStroke = makeToggleCard(rebirthScroll, 5, "Auto Size 2")
    sizeToggle.Activated:Connect(function()
        AutoSize2.active = not AutoSize2.active
        if AutoSize2.active then
            sizeToggle.Text = "ON"; sizeToggle.TextColor3 = T.Text
            sizeToggle.BackgroundColor3 = T.Success
            sizeStroke.Color = T.Success; sizeStroke.Transparency = 0.1
            AutoSize2.lastApply = 0
        else
            sizeToggle.Text = "OFF"; sizeToggle.TextColor3 = T.TextMuted
            sizeToggle.BackgroundColor3 = T.CardDark
            sizeStroke.Color = T.Border; sizeStroke.Transparency = 0.4
        end
    end)

    -- ---------- CONFIG ----------
    local settingsScroll = makeScroll()
    local themeCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 130 or 170), LayoutOrder = 1,
        ZIndex = 5,
    }, settingsScroll)
    UI.corner(themeCard, UDim.new(0, 10))
    Utils.addBtnBg(themeCard, 0.15)
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, themeCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Mudar Cor da Interface", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, themeCard)
    
    local rgbToggleBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "Auto RGB (Combinando com o fundo)",
        TextColor3 = State.rgbMode and Color3.fromRGB(0,0,0) or T.Text,
        TextSize = isMobile and 10 or 12,
        BackgroundColor3 = State.rgbMode and T.Accent or T.CardDark, 
        BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, 30),
        Size = UDim2.new(1, -28, 0, isMobile and 24 or 30),
        ZIndex = 6,
    }, themeCard)
    UI.corner(rgbToggleBtn, UDim.new(0, 6))
    
    local colorsGrid = Utils.create("Frame", {
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,isMobile and 62 or 68),
        Size = UDim2.new(1,-28,0,isMobile and 60 or 85),
        ZIndex = 6,
    }, themeCard)

    local COLORS = {
        { Name="Roxo",     Color=Color3.fromRGB(168, 85, 247) },
        { Name="Azul",     Color=Color3.fromRGB(59, 130, 246) },
        { Name="Verde",    Color=Color3.fromRGB(34, 197, 94) },
        { Name="Rosa",     Color=Color3.fromRGB(236, 72, 153) },
        { Name="Dourado",  Color=Color3.fromRGB(250, 204, 21) },
    }
    
    local function applyAccent(c)
        T.Accent = c
        mainStroke.Color = c
        for _, l in ipairs(accentLabels) do l.TextColor3 = c end
    end

    rgbToggleBtn.Activated:Connect(function()
        State.rgbMode = not State.rgbMode
        if State.rgbMode then
            rgbToggleBtn.TextColor3 = Color3.fromRGB(0,0,0)
            rgbToggleBtn.BackgroundColor3 = T.Accent
        else
            rgbToggleBtn.TextColor3 = T.Text
            rgbToggleBtn.BackgroundColor3 = T.CardDark
            applyAccent(Color3.fromRGB(168, 85, 247))
            mainStroke.Color = Color3.fromRGB(168, 85, 247)
            tabIndicator.BackgroundColor3 = Color3.fromRGB(168, 85, 247)
        end
    end)

    for i, cd in ipairs(COLORS) do
        local x = ((i-1) % 3) * (isMobile and 48 or 60)
        local y = math.floor((i-1) / 3) * (isMobile and 30 or 38)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = cd.Name, TextColor3 = T.Text,
            TextSize = isMobile and 9 or 11,
            BackgroundColor3 = cd.Color, BorderSizePixel = 0,
            AutoButtonColor = false,
            Position = UDim2.new(0,x,0,y),
            Size = UDim2.new(0, isMobile and 44 or 54, 0, isMobile and 24 or 30),
            ZIndex = 6,
        }, colorsGrid)
        UI.corner(b, UDim.new(0, 6))
        Utils.create("UIStroke", {Color = Color3.fromRGB(255,255,255), Thickness = 1, Transparency = 0.7}, b)
        b.Activated:Connect(function()
            State.rgbMode = false
            rgbToggleBtn.TextColor3 = T.Text
            rgbToggleBtn.BackgroundColor3 = T.CardDark
            applyAccent(cd.Color) 
        end)
    end

    local perfCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 125 or 165), LayoutOrder = 2,
        ZIndex = 5,
    }, settingsScroll)
    UI.corner(perfCard, UDim.new(0, 10))
    Utils.addBtnBg(perfCard, 0.15)
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, perfCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Desempenho & Anti-Lag", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, perfCard)

    local function makePerfBtn(lo, label, callback)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamMedium, Text = label, TextColor3 = T.Text,
            TextSize = isMobile and 10 or 12,
            BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
            AutoButtonColor = false,
            Position = UDim2.new(0,14,0,32 + (lo-1)*(isMobile and 28 or 40)),
            Size = UDim2.new(1,-28,0, isMobile and 24 or 34),
            ZIndex = 6,
        }, perfCard)
        UI.corner(b, UDim.new(0, 6))
        b.Activated:Connect(function()
            pcall(callback)
            b.Text = "OK!"; b.TextColor3 = T.Success
            task.wait(2)
            b.Text = label; b.TextColor3 = T.Text
        end)
        return b
    end

    makePerfBtn(1, "Ativar Anti-Lag", function()
        local count = 0
        for _, v in ipairs(workspace:GetDescendants()) do
            count = count + 1
            if count % 300 == 0 then task.wait() end
            pcall(function()
                if v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Smoke")
                or v:IsA("Fire") or v:IsA("Sparkles") then
                    v:Destroy()
                elseif v:IsA("BasePart") then
                    v.Material = Enum.Material.SmoothPlastic
                    v.Reflectance = 0
                end
            end)
        end
    end)
    makePerfBtn(2, "Ativar Fullbright", function()
        Lighting.Brightness = 2; Lighting.ClockTime = 14
        Lighting.FogEnd = 100000; Lighting.GlobalShadows = false
        for _, v in ipairs(Lighting:GetChildren()) do
            if v:IsA("PostEffect") then v.Enabled = false end
        end
    end)
    makePerfBtn(3, "Otimizar Graficos", function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        workspace.CurrentCamera.FieldOfView = 70
    end)

    local TAB_MAP = {
        ["Main"] = mainScroll,
        ["Fast Farming"] = farmScroll,
        ["Rebithers"] = rebirthScroll,
        ["Config"] = settingsScroll,
    }

    local function setTab(name)
        for key, info in pairs(tabInfo) do
            local active = (key == name)
            info.btn.TextColor3 = active and T.Text or T.TextDark
            if TAB_MAP[key] then TAB_MAP[key].Visible = active end
        end
        for i, n in ipairs(tabOrder) do
            if n == name then
                if tabIndicator then
                    tabIndicator.Position = UDim2.new(0, 4 + (i-1)*(TAB_W+TAB_GAP), 1, -3)
                    tabIndicator.Size = UDim2.new(0, TAB_W, 0, 2)
                end
                break
            end
        end
    end

    for key, info in pairs(tabInfo) do
        info.btn.Activated:Connect(function() setTab(key) end)
    end
    setTab("Main")

    task.spawn(function()
        local hue = 0
        while AutoFastGui and AutoFastGui.Parent and Utils.alive() do
            if State.rgbMode then
                pcall(function()
                    hue = (hue + 0.005) % 1
                    local newColor = Color3.fromHSV(hue, 0.75, 0.9)
                    
                    T.Accent = newColor
                    mainStroke.Color = newColor
                    tabIndicator.BackgroundColor3 = newColor
                    rgbToggleBtn.BackgroundColor3 = newColor
                    
                    for _, l in ipairs(accentLabels) do 
                        l.TextColor3 = newColor 
                    end
                end)
            end
            task.wait(0.05)
        end
    end)

    task.delay(0.05, function()
        if not (main and main.Parent) then return end
        TweenService:Create(main,
            TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size = FULL}):Play()
        TweenService:Create(main,
            TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {BackgroundTransparency = 0}):Play()
        TweenService:Create(mainStroke,
            TweenInfo.new(0.4), {Transparency = 0.2}):Play()
    end)
end

--==========================================================
-- 18. MAIN HUB
--==========================================================
function openMainHub()
    if not Utils.alive() then return end
    if MainGui then MainGui:Destroy() end
    State.lastWindow = "main"
    local T = UI.Theme
    local isMobile = Utils.isMobile()
    local FULL = isMobile and Utils.scaledSize(440, 460) or Utils.scaledSize(660, 620)

    if statsGuiRef and statsGuiRef.Parent then
        pcall(function() statsGuiRef:Destroy() end)
    end
    statsGuiRef = nil
    pcall(function()
        local old = PARENT_GUI:FindFirstChild("PS_StatsGui")
        if old then old:Destroy() end
    end)
    pcall(function()
        local old = Player.PlayerGui:FindFirstChild("PS_StatsGui")
        if old then old:Destroy() end
    end)

    MainGui = Utils.create("ScreenGui", {
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true, Name = "PrivateScriptMainGui",
    }, PARENT_GUI)

    local container = Utils.create("Frame", {
        AnchorPoint = Vector2.new(0.5,0.5), BackgroundColor3 = T.Container,
        BackgroundTransparency = 1,
        BorderSizePixel = 0,
        Position = UDim2.new(0.5,0,0.5,0),
        Size = UDim2.new(0,0,0,0),
        ClipsDescendants = true,
    }, MainGui)
    UI.corner(container, UDim.new(0, 14))
    local mainStroke = Utils.create("UIStroke",
        {Color = T.Accent, Thickness = 1.5, Transparency = 1}, container)

    Utils.addUiBackground(container, 0.75)

    local topBar = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,16,0,10),
        Size = UDim2.new(1,-32,0,40),
    }, container)
    Utils.makeDraggable(topBar, container)
    UI.createTitleBar(topBar, isMobile and 12 or 15)

    local contentContainer = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,0,0,58),
        Size = UDim2.new(1,0,1,-58),
    }, container)

    UI.createWindowButtons(topBar,
        function()
            TweenService:Create(container, TweenInfo.new(0.3), {Size = UDim2.new(0,0,0,0)}):Play()
            task.wait(0.35)
            MainGui:Destroy(); MainGui = nil
            if statsGuiRef and statsGuiRef.Parent then
                pcall(function() statsGuiRef:Destroy() end)
            end
            statsGuiRef = nil
            createFloatingButton()
        end,
        function()
            TweenService:Create(container, TweenInfo.new(0.3), {Size = UDim2.new(0,0,0,0)}):Play()
            task.wait(0.35)
            MainGui:Destroy(); MainGui = nil
            if not State.antiAfk.active then
                if statsGuiRef and statsGuiRef.Parent then
                    pcall(function() statsGuiRef:Destroy() end)
                end
                statsGuiRef = nil
            end
            createFloatingButton()
        end,
        function() contentContainer.Visible = true
            TweenService:Create(container, TweenInfo.new(0.3), {Size = FULL}):Play() end
    )

    local tabsArea = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark,
        BorderSizePixel = 0,
        Position = UDim2.new(0,16,0,0),
        Size = UDim2.new(1,-32,0,38),
        ClipsDescendants = true,
    }, contentContainer)
    UI.corner(tabsArea, UDim.new(0, 10))

    local TAB_W = isMobile and 44 or 66
    local TAB_GAP = 12
    local tabInfo = {}
    local tabOrder = {}

    local tabsContainer = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0,4,0,4),
        Size = UDim2.new(0, 9999, 1, -8),
    }, tabsArea)

    local tabIndicator = Utils.create("Frame", {
        BackgroundColor3 = T.Accent,
        BorderSizePixel = 0,
        Position = UDim2.new(0, 4, 1, -3),
        Size = UDim2.new(0, TAB_W, 0, 2),
        ZIndex = 6,
    }, tabsArea)
    UI.corner(tabIndicator, UDim.new(1, 0))

    local function makeTab(i, name)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = name, TextColor3 = T.TextDark,
            TextSize = isMobile and 8 or 10,
            BackgroundColor3 = Color3.fromRGB(0,0,0),
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Position = UDim2.new(0, (i-1)*(TAB_W+TAB_GAP), 0, 0),
            Size = UDim2.new(0, TAB_W, 1, 0),
            AutoButtonColor = false,
        }, tabsContainer)
        UI.corner(b, UDim.new(0, 6))
        tabInfo[name] = { btn = b }
        table.insert(tabOrder, name)

        if i > 1 then
            local sep = Utils.create("Frame", {
                BackgroundColor3 = Color3.fromRGB(255, 255, 255),
                BackgroundTransparency = 1,
                BorderSizePixel = 0,
                AnchorPoint = Vector2.new(1, 0.5),
                Position = UDim2.new(0, -6, 0.5, 0),
                Size = UDim2.new(0, 2, 0, 14),
                ZIndex = 6,
            }, b)

            local sepImg = Utils.create("ImageLabel", {
                BackgroundTransparency = 1,
                Image = CFG.BTN_BG_IMAGE,
                ImageTransparency = 0.3,
                ImageColor3 = T.Border,
                ScaleType = Enum.ScaleType.Stretch,
                Size = UDim2.new(1, 0, 1, 0),
                ZIndex = 6,
            }, sep)

            local gradient = Instance.new("UIGradient")
            gradient.Rotation = 90
            gradient.Transparency = NumberSequence.new({
                NumberSequenceKeypoint.new(0, 1),
                NumberSequenceKeypoint.new(0.2, 0),
                NumberSequenceKeypoint.new(0.8, 0),
                NumberSequenceKeypoint.new(1, 1)
            })
            gradient.Parent = sepImg
            Utils.create("UICorner", {CornerRadius = UDim.new(1, 0)}, sepImg)
        end

        return b
    end

    local TABS = {"Main","Farm","Rebirth","Config","Stats","Teleport","Boss","Halloween","Gift","Fortune","Store"}
    for i, name in ipairs(TABS) do makeTab(i, name) end

    local scrollbarBg = Utils.create("Frame", {
        BackgroundColor3 = T.BorderSoft, BorderSizePixel = 0,
        Position = UDim2.new(0,16,0,44), Size = UDim2.new(1,-32,0,3),
        Active = false,
    }, contentContainer)
    UI.corner(scrollbarBg, UDim.new(1,0))

    local scrollThumb = Utils.create("Frame", {
        BackgroundColor3 = T.Accent, BorderSizePixel = 0,
        Position = UDim2.new(0, 0, 0, 0), Size = UDim2.new(0.2, 0, 1, 0),
        ZIndex = 5, Active = true,
    }, scrollbarBg)
    UI.corner(scrollThumb, UDim.new(1,0))

    local dragScrollOffset = 0

    local function scrollTabsTo(index)
        local maxIndex = #tabOrder
        index = math.clamp(index, 1, maxIndex)
        local totalWidth = maxIndex * (TAB_W + TAB_GAP)
        local areaW = tabsArea.AbsoluteSize.X
        local maxScroll = math.max(0, totalWidth - areaW)
        local tabCenter = (index - 1) * (TAB_W + TAB_GAP) + TAB_W / 2
        local newScroll = math.clamp(tabCenter - areaW / 2, 0, maxScroll)
        dragScrollOffset = newScroll
        tabsContainer.Position = UDim2.new(0, -newScroll, 0, 0)
        local thumbSizeX = math.max(0.15, areaW / totalWidth)
        local thumbX = maxScroll > 0 and (newScroll / maxScroll) * (1 - thumbSizeX) or 0
        scrollThumb.Position = UDim2.new(thumbX, 0, 0, 0)
        scrollThumb.Size = UDim2.new(thumbSizeX, 0, 1, 0)
    end

    local dragging = false
    local dragStartX, dragStartScrollX

    local thumbHitbox = Utils.create("Frame", {
        BackgroundTransparency = 1, BorderSizePixel = 0,
        Position = UDim2.new(0, 0, 0, -8), Size = UDim2.new(1, 0, 1, 16),
        ZIndex = 6, Active = true,
    }, scrollThumb)

    thumbHitbox.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStartX = input.Position.X
            dragStartScrollX = dragScrollOffset
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch) then
            local dx = input.Position.X - dragStartX
            local bgW = scrollbarBg.AbsoluteSize.X
            local totalWidth = #tabOrder * (TAB_W + TAB_GAP)
            local areaW = tabsArea.AbsoluteSize.X
            local maxScroll = math.max(0, totalWidth - areaW)
            dragScrollOffset = math.clamp(dragStartScrollX + (dx / math.max(bgW, 1)) * maxScroll, 0, maxScroll)
            tabsContainer.Position = UDim2.new(0, -dragScrollOffset, 0, 0)
            if maxScroll > 0 then
                local thumbSizeX = math.max(0.15, areaW / totalWidth)
                local thumbX = (dragScrollOffset / maxScroll) * (1 - thumbSizeX)
                scrollThumb.Position = UDim2.new(thumbX, 0, 0, 0)
                scrollThumb.Size = UDim2.new(thumbSizeX, 0, 1, 0)
            end
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    local contentArea = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark,
        BorderSizePixel = 0,
        Position = UDim2.new(0,16,0,60),
        Size = UDim2.new(1,-32,1,-76),
        ClipsDescendants = true,
        ZIndex = 1,
    }, contentContainer)
    UI.corner(contentArea, UDim.new(0, 10))
    Utils.create("UIStroke",
        {Color = T.Border, Thickness = 1, Transparency = 0.4}, contentArea)

    local accentLabels = {}
    local function makeScroll()
        local s = Utils.create("ScrollingFrame", {
            BackgroundTransparency = 1,
            Position = UDim2.new(0,10,0,10),
            Size = UDim2.new(1,-20,1,-20),
            CanvasSize = UDim2.new(0,0,0,2000),
            ScrollBarThickness = 4,
            ScrollBarImageColor3 = T.Accent,
            ScrollBarImageTransparency = 0.3,
            Visible = false,
            ZIndex = 5,
        }, contentArea)
        Utils.create("UIListLayout",
            {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 14)}, s)
        Utils.create("UIPadding",
            {PaddingTop = UDim.new(0,4), PaddingBottom = UDim.new(0,20),
             PaddingLeft = UDim.new(0,4), PaddingRight = UDim.new(0,4)}, s)
        return s
    end

    local CARD_H = isMobile and 44 or 62

    local function makeCard(parent, order, height)
        local wrap = Utils.create("Frame", {
            BackgroundTransparency = 1,
            BorderSizePixel = 0,
            Size = UDim2.new(1, 0, 0, (height or CARD_H)),
            LayoutOrder = order,
            ZIndex = 5,
        }, parent)

        local card = Utils.create("Frame", {
            BackgroundColor3 = T.Card,
            BackgroundTransparency = 0.35,
            BorderSizePixel = 0,
            Size = UDim2.new(1,0,0, height or CARD_H),
            LayoutOrder = 0,
            ZIndex = 5,
        }, wrap)
        UI.corner(card, UDim.new(0, 10))
        Utils.addBtnBg(card, 0.15)
        Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, card)
        return card
    end

    -- ---------- MAIN ----------
    local mainScroll = makeScroll()
    mainScroll.Visible = true

    local devCard = makeCard(mainScroll, 1)
    local devTitle = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "DEVELOPER INFO", TextColor3 = T.Accent,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,6),
        Size = UDim2.new(1,-28,0,14), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, devCard)
    table.insert(accentLabels, devTitle)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "Criador / DEVELOP: Kil09122", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13, BackgroundTransparency = 1,
        Position = UDim2.new(0,14,0,22),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, devCard)

    local antiCard = makeCard(mainScroll, 3)
    local antiStroke = Utils.create("UIStroke",
        {Color = State.antiAfk.active and T.Success or T.Border,
         Thickness = State.antiAfk.active and 2 or 1,
         Transparency = State.antiAfk.active and 0.1 or 0.4}, antiCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "ANTI AFK", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, antiCard)
    local antiBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = State.antiAfk.active and "ON" or "OFF",
        TextColor3 = State.antiAfk.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = State.antiAfk.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 76 or 100, 0, isMobile and 28 or 36),
        AutoButtonColor = false, ZIndex = 6,
    }, antiCard)
    UI.corner(antiBtn, UDim.new(1,0))
    Utils.create("UIStroke", {
        Color = State.antiAfk.active and T.Success or T.Border,
        Thickness = 1, Transparency = 0.3,
    }, antiBtn)

    statsGuiRef = Utils.create("ScreenGui", {
        ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
        IgnoreGuiInset = true, Name = "PS_StatsGui",
        Enabled = State.antiAfk.active,
    }, PARENT_GUI)
    local statsGui = statsGuiRef

    local statsBox = Utils.create("Frame", {
        BackgroundColor3 = T.Container, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Position = UDim2.new(0, 40, 0, 40), Size = UDim2.new(0, 260, 0, 172),
        ClipsDescendants = true,
    }, statsGui)
    UI.corner(statsBox, UDim.new(0, 12))
    Utils.addBtnBg(statsBox, 0.3)
    Utils.create("UIStroke", {Color = T.Accent, Thickness = 1.5, Transparency = 0.3}, statsBox)
    Utils.makeDraggable(statsBox, statsBox)

    statsBox.Size = UDim2.new(0, 0, 0, 0)
    TweenService:Create(statsBox, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Size = UDim2.new(0, 260, 0, 172)
    }):Play()

    local headerRow = Utils.create("Frame", {
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 12, 0, 10),
        Size = UDim2.new(1, -24, 0, 22),
        ZIndex = 6,
    }, statsBox)

    local pumpkin = Utils.create("ImageLabel", {
        BackgroundTransparency = 1,
        Image = CFG.BANNER_IMAGE,
        ScaleType = Enum.ScaleType.Crop,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 0, 0.5, 0),
        Size = UDim2.new(0, 20, 0, 20),
        ZIndex = 6,
    }, headerRow)
    UI.corner(pumpkin, UDim.new(1, 0))
    Utils.create("UIStroke", {
        Color = T.Accent, Thickness = 1.5, Transparency = 0.2,
    }, pumpkin)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "ANTI AFK - STATUS", TextColor3 = T.Accent,
        TextSize = 12, BackgroundTransparency = 1,
        Position = UDim2.new(0, 26, 0, 0),
        Size = UDim2.new(1, -50, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, headerRow)

    local minAntiBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "—",
        TextColor3 = T.TextMuted, TextSize = 14,
        BackgroundColor3 = T.CardDark,
        BorderSizePixel = 0,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, 0, 0.5, 0),
        Size = UDim2.new(0, 20, 0, 20),
        AutoButtonColor = false,
        ZIndex = 7,
    }, headerRow)
    UI.corner(minAntiBtn, UDim.new(1,0))
    minAntiBtn.Activated:Connect(function()
        TweenService:Create(statsBox, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Size = UDim2.new(0, 0, 0, 0)
        }):Play()
        task.wait(0.3)
        if statsGuiRef and statsGuiRef.Parent then
            statsGuiRef.Enabled = false
        end
        State.antiAfk.isOpen = false
    end)

    Utils.create("Frame", {
        BackgroundColor3 = T.BorderSoft, BorderSizePixel = 0,
        Position = UDim2.new(0,14,0,38), Size = UDim2.new(1,-28,0,1),
        ZIndex = 6,
    }, statsBox)
    local statImg = Utils.create("ImageLabel", {
        BackgroundColor3 = T.Card, BorderSizePixel = 0,
        Position = UDim2.new(0,14,0,50), Size = UDim2.new(0, 56, 0, 56),
        Image = Utils.getPlayerImage(), ScaleType = Enum.ScaleType.Crop,
        ZIndex = 6,
    }, statsBox)
    UI.corner(statImg, UDim.new(0, 10))
    Utils.create("UIStroke", {Color = T.Success, Thickness = 2, Transparency = 0.4}, statImg)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = Player.Name,
        TextColor3 = T.Text, TextSize = 12, BackgroundTransparency = 1,
        Position = UDim2.new(0,82,0,50), Size = UDim2.new(1,-96,0,20),
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
    }, statsBox)

    local fpsLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "FPS: --", TextColor3 = T.Success,
        TextSize = 11, BackgroundTransparency = 1, Position = UDim2.new(0,82,0,70),
        Size = UDim2.new(1,-96,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, statsBox)
    local pingLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "Ping: -- ms",
        TextColor3 = T.Gold, TextSize = 11,
        BackgroundTransparency = 1, Position = UDim2.new(0,82,0,88),
        Size = UDim2.new(1,-96,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, statsBox)
    local uptimeLbl = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Tempo: 00:00:00",
        TextColor3 = T.Welcome, TextSize = 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,120),
        Size = UDim2.new(1,-28,0,22), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, statsBox)

    task.spawn(function()
        while Utils.alive() do
            pcall(function()
                if State.antiAfk.active and statsGui and statsGui.Parent then
                    local fps = math.floor(60 * (1 / math.max(workspace:GetRealPhysicsFPS(), 1)))
                    fpsLbl.Text = "FPS: " .. tostring(fps)
                    local pv = 0
                    pcall(function() pv = math.round(Player:GetNetworkPing() * 1000) end)
                    pingLbl.Text = "Ping: " .. tostring(pv) .. " ms"
                    local elapsed = math.floor(tick() - State.antiAfk.startTime + State.antiAfk.totalTime)
                    uptimeLbl.Text = string.format("Tempo: %02d:%02d:%02d",
                        math.floor(elapsed/3600), math.floor((elapsed%3600)/60), elapsed%60)
                end
            end)
            task.wait(1)
        end
    end)

    if not idledConnected then
        idledConnected = true
        Player.Idled:Connect(function()
            if State.antiAfk.active then
                pcall(function()
                    VirtualUser:CaptureController()
                    VirtualUser:ClickButton2(Vector2.new())
                end)
            end
        end)
    end

    antiBtn.Activated:Connect(function()
        if State.antiAfk.active then
            State.antiAfk.active = false
            State.antiAfk.totalTime = 0
            State.antiAfk.startTime = 0
            if statsGui and statsGui.Parent then statsGui.Enabled = false end
            antiBtn.Text = "OFF"; antiBtn.TextColor3 = T.TextMuted
            antiBtn.BackgroundColor3 = T.CardDark
            antiStroke.Color = T.Border; antiStroke.Transparency = 0.4
        else
            State.antiAfk.active = true
            State.antiAfk.startTime = tick()
            State.antiAfk.totalTime = 0
            if statsGui and statsGui.Parent then statsGui.Enabled = true end
            antiBtn.Text = "ON"; antiBtn.TextColor3 = T.Text
            antiBtn.BackgroundColor3 = T.Success
            antiStroke.Color = T.Success; antiStroke.Transparency = 0.1
            
            if statsGui and statsGui.Parent then
                local box = statsGui:FindFirstChildOfClass("Frame")
                if box then
                    box.Size = UDim2.new(0, 0, 0, 0)
                    TweenService:Create(box, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
                        Size = UDim2.new(0, 260, 0, 172)
                    }):Play()
                end
            end
        end
    end)

    local updCard = makeCard(mainScroll, 4)
    local updTitle = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "ATUALIZAÇÕES", TextColor3 = T.Accent,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,6),
        Size = UDim2.new(1,-28,0,14), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, updCard)
    table.insert(accentLabels, updTitle)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium,
        Text = "v18.12.15 - Detector de Força Moderno",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12, TextWrapped = true,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,22),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Top, ZIndex = 6,
    }, updCard)

    local discordBtn = Utils.create("TextButton", {
        BackgroundColor3 = T.Discord, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 38 or 46), LayoutOrder = 5,
        AutoButtonColor = false, Text = "", ZIndex = 5,
    }, mainScroll)
    UI.corner(discordBtn, UDim.new(0, 10))
    Utils.addBtnBg(discordBtn, 0.5)
    Utils.create("UIStroke", {Color = T.DiscordHi, Thickness = 1, Transparency = 0.5}, discordBtn)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Copiar Link do Discord", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13, BackgroundTransparency = 1,
        Position = UDim2.new(0,44,0,0),
        Size = UDim2.new(1,-52,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, discordBtn)

    local discordIcon = Utils.create("ImageLabel", {
        BackgroundTransparency = 1,
        Image = CFG.DISCORD_ICON,
        ScaleType = Enum.ScaleType.Crop,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 12, 0.5, 0),
        Size = UDim2.new(0, 22, 0, 22),
        ZIndex = 6,
    }, discordBtn)
    UI.corner(discordIcon, UDim.new(1, 0))

    discordBtn.MouseEnter:Connect(function()
        TweenService:Create(discordBtn, TweenInfo.new(0.2), {BackgroundColor3 = T.DiscordHi}):Play()
    end)
    discordBtn.MouseLeave:Connect(function()
        TweenService:Create(discordBtn, TweenInfo.new(0.2), {BackgroundColor3 = T.Discord}):Play()
    end)
    discordBtn.Activated:Connect(function()
        if setclipboard then setclipboard("https://discord.gg/WRphBR4AV") end
    end)

    -- ==========================================================
    -- ABA STATS
    -- ==========================================================
    local statsScroll = makeScroll()
    local selPlayer = Player
    local isListOpen = false

    local toggleListBtn = Utils.create("TextButton", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1, 0, 0, 42),
        LayoutOrder = 1, ZIndex = 5,
        Text = "", AutoButtonColor = false,
    }, statsScroll)
    UI.corner(toggleListBtn, UDim.new(0, 10))
    Utils.addBtnBg(toggleListBtn, 0.15)
    local toggleStroke = Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, toggleListBtn)
    
    local toggleText = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Selecionar Jogador",
        TextColor3 = T.Text, TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Position = UDim2.new(0, 14, 0, 0),
        Size = UDim2.new(1, -44, 1, 0), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, toggleListBtn)
    
    local arrowIcon = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "▼",
        TextColor3 = T.Accent, TextSize = 12,
        BackgroundTransparency = 1, AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -14, 0.5, 0), Size = UDim2.new(0, 20, 0, 20),
        ZIndex = 6,
    }, toggleListBtn)

    local listFrame = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
        Size = UDim2.new(1, 0, 0, 0),
        LayoutOrder = 2, ZIndex = 5, ClipsDescendants = true,
    }, statsScroll)
    UI.corner(listFrame, UDim.new(0, 10))
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, listFrame)

    local listScroll = Utils.create("ScrollingFrame", {
        BackgroundTransparency = 1,
        Size = UDim2.new(1, -10, 1, -10),
        Position = UDim2.new(0, 5, 0, 5),
        CanvasSize = UDim2.new(0, 0, 0, 0),
        ScrollBarThickness = 3, ScrollBarImageColor3 = T.Accent,
        ZIndex = 6,
    }, listFrame)
    Utils.create("UIListLayout", {SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 4)}, listScroll)

    local function refreshPlayerList()
        for _, child in ipairs(listScroll:GetChildren()) do
            if child:IsA("TextButton") then
                child:Destroy()
            end
        end

        local players = Players:GetPlayers()
        table.sort(players, function(a, b) return a.Name:lower() < b.Name:lower() end)

        for i, plr in ipairs(players) do
            local btn = Utils.create("TextButton", {
                BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
                Size = UDim2.new(1, 0, 0, 32), LayoutOrder = i,
                Text = plr.Name, TextColor3 = T.Text,
                Font = Enum.Font.GothamBold, TextSize = 12,
                AutoButtonColor = false,
                ZIndex = 6,
            }, listScroll)
            UI.corner(btn, UDim.new(0, 6))
            Utils.addBtnBg(btn, 0.2)
            
            if plr == selPlayer then
                btn.BackgroundColor3 = T.Success
            end

            btn.MouseEnter:Connect(function()
                if plr ~= selPlayer then
                    TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = T.CardHover}):Play()
                end
            end)
            btn.MouseLeave:Connect(function()
                if plr ~= selPlayer then
                    TweenService:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = T.Card}):Play()
                end
            end)

            btn.Activated:Connect(function()
                selPlayer = plr
                refreshPlayerList()
                toggleText.Text = "Selecionado: " .. selPlayer.Name
                
                isListOpen = false
                listFrame.Size = UDim2.new(1, 0, 0, 0)
                arrowIcon.Rotation = 0
            end)
        end
        
        local contentHeight = #players * 36
        listScroll.CanvasSize = UDim2.new(0, 0, 0, contentHeight)
    end

    toggleListBtn.Activated:Connect(function()
        isListOpen = not isListOpen
        if isListOpen then
            refreshPlayerList()
            local playersCount = #Players:GetPlayers()
            local targetHeight = math.clamp(playersCount * 36 + 10, 44, 160)
            listFrame.Size = UDim2.new(1, 0, 0, targetHeight)
            arrowIcon.Rotation = 180
        else
            listFrame.Size = UDim2.new(1, 0, 0, 0)
            arrowIcon.Rotation = 0
        end
    end)

    Players.PlayerAdded:Connect(function() if isListOpen then refreshPlayerList() end end)
    Players.PlayerRemoving:Connect(function(plr)
        if selPlayer == plr then
            selPlayer = Player
            toggleText.Text = "Selecionado: " .. selPlayer.Name
        end
        if isListOpen then refreshPlayerList() end
    end)

    toggleText.Text = "Selecionado: " .. selPlayer.Name

    local statsTitle = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "Stats Player",
        TextColor3 = T.Text,
        TextSize = isMobile and 14 or 17,
        BackgroundTransparency = 1,
        Size = UDim2.new(1, 0, 0, 28),
        LayoutOrder = 3,
        TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 5,
    }, statsScroll)
    Utils.create("UIPadding", {PaddingTop = UDim.new(0, 10), PaddingLeft = UDim.new(0, 4)}, statsTitle)

    local function createStatCard(order, titulo, cor, iconeId)
        local card = makeCard(statsScroll, order)
        Utils.create("UIStroke", {Color = cor, Thickness = 1.5, Transparency = 0.4}, card)

        local icon = Utils.create("ImageLabel", {
            BackgroundTransparency = 1, Image = iconeId,
            ImageColor3 = cor,
            ScaleType = Enum.ScaleType.Fit, AnchorPoint = Vector2.new(0, 0.5),
            Position = UDim2.new(0, 12, 0.5, 0),
            Size = UDim2.new(0, isMobile and 26 or 32, 0, isMobile and 26 or 32),
            ZIndex = 6,
        }, card)

        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = titulo,
            TextColor3 = T.TextMuted,
            TextSize = isMobile and 11 or 13, BackgroundTransparency = 1,
            Position = UDim2.new(0, isMobile and 46 or 54, 0, 0),
            Size = UDim2.new(0.5, 0, 1, 0),
            TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, card)

        local val = Utils.create("TextLabel", {
            Font = Enum.Font.GothamBlack, Text = "0",
            TextColor3 = cor, TextSize = isMobile and 13 or 17,
            BackgroundTransparency = 1, AnchorPoint = Vector2.new(1, 0.5),
            Position = UDim2.new(1, -14, 0.5, 0),
            Size = UDim2.new(0.5, -14, 1, 0),
            TextXAlignment = Enum.TextXAlignment.Right,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, card)
        return val
    end

    local strengthValue = createStatCard(4, "Força", Color3.fromRGB(248, 113, 113), CFG.LAUNCHER_ICON_ID)
    local durabilityValue = createStatCard(5, "Durabilidade", Color3.fromRGB(96, 165, 250), CFG.LAUNCHER_ICON_ID)
    local agilityValue = createStatCard(6, "Agilidade", Color3.fromRGB(52, 211, 153), CFG.LAUNCHER_ICON_ID)
    local rebirthsValue = createStatCard(7, "Renascimentos", Color3.fromRGB(251, 146, 60), CFG.LAUNCHER_ICON_ID)
    local brawlsValue = createStatCard(8, "Batalhas", Color3.fromRGB(236, 72, 153), CFG.LAUNCHER_ICON_ID)

    task.spawn(function()
        while MainGui and MainGui.Parent and Utils.alive() do
            pcall(function()
                if selPlayer and selPlayer.Parent then
                    local ls = selPlayer:FindFirstChild("leaderstats")
                    
                    local str = ls and ls:FindFirstChild("Strength")
                    strengthValue.Text = str and Utils.formatNumber(str.Value) or "0"

                    local dur = selPlayer:FindFirstChild("Durability")
                    if dur and dur:IsA("ValueBase") then durabilityValue.Text = Utils.formatNumber(dur.Value)
                    elseif dur and type(dur) == "number" then durabilityValue.Text = Utils.formatNumber(dur)
                    else durabilityValue.Text = "0" end

                    local agi = selPlayer:FindFirstChild("Agility")
                    if agi and agi:IsA("ValueBase") then agilityValue.Text = Utils.formatNumber(agi.Value)
                    elseif agi and type(agi) == "number" then agilityValue.Text = Utils.formatNumber(agi)
                    else agilityValue.Text = "0" end

                    local reb = ls and ls:FindFirstChild("Rebirths")
                    if reb and reb:IsA("ValueBase") then rebirthsValue.Text = Utils.formatNumber(reb.Value)
                    elseif reb and type(reb) == "number" then rebirthsValue.Text = Utils.formatNumber(reb)
                    else rebirthsValue.Text = "0" end

                    local brawls = ls and ls:FindFirstChild("Brawls")
                    if brawls and brawls:IsA("ValueBase") then brawlsValue.Text = Utils.formatNumber(brawls.Value)
                    elseif brawls and type(brawls) == "number" then brawlsValue.Text = Utils.formatNumber(brawls)
                    else brawlsValue.Text = "0" end
                else
                    selPlayer = Player
                    toggleText.Text = "Selecionado: " .. selPlayer.Name
                end
            end)
            task.wait(0.25)
        end
    end)

    -- ---------- FARM ----------
    local farmScroll = makeScroll()
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "AUTO (peso/treino)", TextColor3 = T.Accent,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Size = UDim2.new(1,0,0,18),
        LayoutOrder = 1, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, farmScroll)

    local function createFarmCard(lo, label, key, isFast)
        local field = isFast and "fast" or "auto"
        local isOn = State.tools[key][field]
        local c = makeCard(farmScroll, lo, isMobile and 38 or 54)
        Utils.create("UIStroke", {
            Color = isFast and T.FastBlue or T.Border,
            Thickness = 1, Transparency = 0.4,
        }, c)
        Utils.create("TextLabel", {
            Font = Enum.Font.GothamMedium, Text = label, TextColor3 = T.Text,
            TextSize = isMobile and 10 or 12,
            BackgroundTransparency = 1, Position = UDim2.new(0,14,0,0),
            Size = UDim2.new(1,-80,1,0), TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, c)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = isOn and "ON" or "OFF",
            TextColor3 = isOn and T.Text or T.TextMuted,
            TextSize = isMobile and 10 or 11,
            BackgroundColor3 = isOn and T.Success or T.CardDark,
            BorderSizePixel = 0, AnchorPoint = Vector2.new(1,0.5),
            Position = UDim2.new(1,-12,0.5,0),
            Size = UDim2.new(0, isMobile and 52 or 60, 0, isMobile and 22 or 28),
            AutoButtonColor = false, ZIndex = 6,
        }, c)
        UI.corner(b, UDim.new(1,0))
        b.Activated:Connect(function()
            State.tools[key][field] = not State.tools[key][field]
            if State.tools[key][field] then
                b.Text = "ON"; b.TextColor3 = T.Text
                b.BackgroundColor3 = T.Success
            else
                b.Text = "OFF"; b.TextColor3 = T.TextMuted
                b.BackgroundColor3 = T.CardDark
            end
        end)
    end

    createFarmCard(2, "Auto Weight / Auto Lift", "Weight", false)
    createFarmCard(3, "Auto Situps", "Situps", false)
    createFarmCard(4, "Auto Pushups", "Pushups", false)
    createFarmCard(5, "Auto Handstands", "Handstands", false)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "TOOL FAST", TextColor3 = T.Accent,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Size = UDim2.new(1,0,0,18),
        LayoutOrder = 6, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, farmScroll)
    createFarmCard(7, "Weight - Tool Fast", "Weight", true)
    createFarmCard(8, "Situps - Tool Fast", "Situps", true)
    createFarmCard(9, "Pushups - Tool Fast", "Pushups", true)
    createFarmCard(10, "Handstands - Tool Fast", "Handstands", true)

    task.spawn(function()
        while Utils.alive() do
            pcall(function()
                local char = Player.Character
                if char and not AutoFast.active and not State.autoBossFarm.active then
                    local hum = char:FindFirstChildOfClass("Humanoid")
                    if hum then
                        if State.tools.Weight.auto or State.tools.Weight.fast then
                            local bp = Player:FindFirstChild("Backpack")
                            local ct = char:FindFirstChildOfClass("Tool")
                            local isW = ct and not Anim.isPunchTool(ct)
                                and (Utils.lower(ct.Name):find("weight")
                                  or Utils.lower(ct.Name):find("peso")
                                  or Utils.lower(ct.Name):find("dumbbell")
                                  or Utils.lower(ct.Name):find("barbell")
                                  or Utils.lower(ct.Name):find("lift"))
                            if not isW and bp then
                                for _, i in ipairs(bp:GetChildren()) do
                                    if i:IsA("Tool") and not Anim.isPunchTool(i) then
                                        local n = Utils.lower(i.Name)
                                        if n:find("weight") or n:find("peso")
                                        or n:find("dumbbell") or n:find("barbell")
                                        or n:find("lift") then
                                            hum:EquipTool(i); ct = i; break
                                        end
                                    end
                                end
                            end
                            if ct and not Anim.isPunchTool(ct) then ct:Activate() end
                        end
                        for _, tn in ipairs({"Situps","Pushups","Handstands"}) do
                            if State.tools[tn].auto or State.tools[tn].fast then
                                local bp = Player:FindFirstChild("Backpack")
                                local t = char:FindFirstChild(tn) or (bp and bp:FindFirstChild(tn))
                                if t and t.Parent ~= char then
                                    hum:EquipTool(t); task.wait(0.05)
                                end
                                if t then t:Activate() end
                            end
                        end
                    end
                end
            end)
            task.wait(0.1)
        end
    end)

    -- ---------- REBIRTH ----------
    local rebirthScroll = makeScroll()
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Rebirths", TextColor3 = T.Rebirth,
        TextSize = isMobile and 12 or 15,
        BackgroundTransparency = 1, Size = UDim2.new(1,0,0,20),
        LayoutOrder = 1, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, rebirthScroll)

    local function makeToggleCard(parent, order, labelText)
        local card = makeCard(parent, order)
        local stroke = Utils.create("UIStroke",
            {Color = T.Border, Thickness = 1, Transparency = 0.4}, card)
        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = labelText, TextColor3 = T.Text,
            TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
            Position = UDim2.new(0,14,0.5,0),
            Size = UDim2.new(1,-140,1,0), TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, card)
        local btn = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = "OFF",
            TextColor3 = T.TextMuted, TextSize = isMobile and 11 or 13,
            BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
            AnchorPoint = Vector2.new(1,0.5),
            Position = UDim2.new(1,-14,0.5,0),
            Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
            AutoButtonColor = false, ZIndex = 6,
        }, card)
        UI.corner(btn, UDim.new(1,0))
        return card, btn, stroke
    end

    local rebirthCard, rebirthToggle, rebirthStroke = makeToggleCard(rebirthScroll, 2, "Auto Rebirth")
    rebirthToggle.Activated:Connect(function()
        AutoRebirth.active = not AutoRebirth.active
        if AutoRebirth.active then
            AutoRebirth.lastFire = 0
            rebirthToggle.Text = "ON"; rebirthToggle.TextColor3 = T.Text
            rebirthToggle.BackgroundColor3 = T.Success
            rebirthStroke.Color = T.Success; rebirthStroke.Transparency = 0.1
        else
            rebirthToggle.Text = "OFF"; rebirthToggle.TextColor3 = T.TextMuted
            rebirthToggle.BackgroundColor3 = T.CardDark
            rebirthStroke.Color = T.Border; rebirthStroke.Transparency = 0.4
        end
    end)

    local tpCard, tpToggle, tpStroke = makeToggleCard(rebirthScroll, 3, "Auto TP Muscle King")
    tpToggle.Activated:Connect(function()
        State.autoTP.active = not State.autoTP.active
        AutoTP.active = State.autoTP.active
        if State.autoTP.active then
            AutoTP.lastTP = 0
            tpToggle.Text = "ON"; tpToggle.TextColor3 = T.Text
            tpToggle.BackgroundColor3 = T.Success
            tpStroke.Color = T.Success; tpStroke.Transparency = 0.1
            State.autoTP.keepLock = true
            State.lockPos.active = true
            State.lockPos.pos = AutoTP.customPos
        else
            tpToggle.Text = "OFF"; tpToggle.TextColor3 = T.TextMuted
            tpToggle.BackgroundColor3 = T.CardDark
            tpStroke.Color = T.Border; tpStroke.Transparency = 0.4
            State.autoTP.keepLock = false
            State.lockPos.active = false
        end
    end)

    local lockCard, lockToggle, lockStroke = makeToggleCard(rebirthScroll, 4, "Lock Position")
    lockToggle.Activated:Connect(function()
        State.lockPos.active = not State.lockPos.active
        if State.lockPos.active then
            local char = Player.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            State.lockPos.pos = hrp and hrp.Position or nil
            lockToggle.Text = "ON"; lockToggle.TextColor3 = T.Text
            lockToggle.BackgroundColor3 = T.Success
            lockStroke.Color = T.Success; lockStroke.Transparency = 0.1
        else
            State.lockPos.pos = nil
            lockToggle.Text = "OFF"; lockToggle.TextColor3 = T.TextMuted
            lockToggle.BackgroundColor3 = T.CardDark
            lockStroke.Color = T.Border; lockStroke.Transparency = 0.4
        end
    end)

    local sizeCard, sizeToggle, sizeStroke = makeToggleCard(rebirthScroll, 5, "Auto Size 2")
    sizeToggle.Activated:Connect(function()
        AutoSize2.active = not AutoSize2.active
        if AutoSize2.active then
            sizeToggle.Text = "ON"; sizeToggle.TextColor3 = T.Text
            sizeToggle.BackgroundColor3 = T.Success
            sizeStroke.Color = T.Success; sizeStroke.Transparency = 0.1
            AutoSize2.lastApply = 0
        else
            sizeToggle.Text = "OFF"; sizeToggle.TextColor3 = T.TextMuted
            sizeToggle.BackgroundColor3 = T.CardDark
            sizeStroke.Color = T.Border; sizeStroke.Transparency = 0.4
        end
    end)

    -- ---------- CONFIG ----------
    local settingsScroll = makeScroll()
    local themeCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 130 or 170), LayoutOrder = 1,
        ZIndex = 5,
    }, settingsScroll)
    UI.corner(themeCard, UDim.new(0, 10))
    Utils.addBtnBg(themeCard, 0.15)
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, themeCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Mudar Cor da Interface", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, themeCard)
    
    local rgbToggleBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "Auto RGB (Combinando com o fundo)",
        TextColor3 = State.rgbMode and Color3.fromRGB(0,0,0) or T.Text,
        TextSize = isMobile and 10 or 12,
        BackgroundColor3 = State.rgbMode and T.Accent or T.CardDark, 
        BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, 30),
        Size = UDim2.new(1, -28, 0, isMobile and 24 or 30),
        ZIndex = 6,
    }, themeCard)
    UI.corner(rgbToggleBtn, UDim.new(0, 6))
    
    local colorsGrid = Utils.create("Frame", {
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,isMobile and 62 or 68),
        Size = UDim2.new(1,-28,0,isMobile and 60 or 85),
        ZIndex = 6,
    }, themeCard)

    local COLORS = {
        { Name="Roxo",     Color=Color3.fromRGB(168, 85, 247) },
        { Name="Azul",     Color=Color3.fromRGB(59, 130, 246) },
        { Name="Verde",    Color=Color3.fromRGB(34, 197, 94) },
        { Name="Rosa",     Color=Color3.fromRGB(236, 72, 153) },
        { Name="Dourado",  Color=Color3.fromRGB(250, 204, 21) },
    }
    local function applyAccent(c)
        T.Accent = c
        mainStroke.Color = c
        for _, l in ipairs(accentLabels) do l.TextColor3 = c end
    end

    rgbToggleBtn.Activated:Connect(function()
        State.rgbMode = not State.rgbMode
        if State.rgbMode then
            rgbToggleBtn.TextColor3 = Color3.fromRGB(0,0,0)
            rgbToggleBtn.BackgroundColor3 = T.Accent
        else
            rgbToggleBtn.TextColor3 = T.Text
            rgbToggleBtn.BackgroundColor3 = T.CardDark
            applyAccent(Color3.fromRGB(168, 85, 247))
            mainStroke.Color = Color3.fromRGB(168, 85, 247)
            tabIndicator.BackgroundColor3 = Color3.fromRGB(168, 85, 247)
        end
    end)

    for i, cd in ipairs(COLORS) do
        local x = ((i-1) % 3) * (isMobile and 48 or 60)
        local y = math.floor((i-1) / 3) * (isMobile and 30 or 38)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = cd.Name, TextColor3 = T.Text,
            TextSize = isMobile and 9 or 11,
            BackgroundColor3 = cd.Color, BorderSizePixel = 0,
            AutoButtonColor = false,
            Position = UDim2.new(0,x,0,y),
            Size = UDim2.new(0, isMobile and 44 or 54, 0, isMobile and 24 or 30),
            ZIndex = 6,
        }, colorsGrid)
        UI.corner(b, UDim.new(0, 6))
        Utils.create("UIStroke", {Color = Color3.fromRGB(255,255,255), Thickness = 1, Transparency = 0.7}, b)
        b.Activated:Connect(function()
            State.rgbMode = false
            rgbToggleBtn.TextColor3 = T.Text
            rgbToggleBtn.BackgroundColor3 = T.CardDark
            applyAccent(cd.Color) 
        end)
    end

    local perfCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 125 or 165), LayoutOrder = 2,
        ZIndex = 5,
    }, settingsScroll)
    UI.corner(perfCard, UDim.new(0, 10))
    Utils.addBtnBg(perfCard, 0.15)
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, perfCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Desempenho & Anti-Lag", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, perfCard)

    local function makePerfBtn(lo, label, callback)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamMedium, Text = label, TextColor3 = T.Text,
            TextSize = isMobile and 10 or 12,
            BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
            AutoButtonColor = false,
            Position = UDim2.new(0,14,0,32 + (lo-1)*(isMobile and 28 or 40)),
            Size = UDim2.new(1,-28,0, isMobile and 24 or 34),
            ZIndex = 6,
        }, perfCard)
        UI.corner(b, UDim.new(0, 6))
        b.Activated:Connect(function()
            pcall(callback)
            b.Text = "OK!"; b.TextColor3 = T.Success
            task.wait(2)
            b.Text = label; b.TextColor3 = T.Text
        end)
        return b
    end

    makePerfBtn(1, "Ativar Anti-Lag", function()
        local count = 0
        for _, v in ipairs(workspace:GetDescendants()) do
            count = count + 1
            if count % 300 == 0 then task.wait() end
            pcall(function()
                if v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Smoke")
                or v:IsA("Fire") or v:IsA("Sparkles") then
                    v:Destroy()
                elseif v:IsA("BasePart") then
                    v.Material = Enum.Material.SmoothPlastic
                    v.Reflectance = 0
                end
            end)
        end
    end)
    makePerfBtn(2, "Ativar Fullbright", function()
        Lighting.Brightness = 2; Lighting.ClockTime = 14
        Lighting.FogEnd = 100000; Lighting.GlobalShadows = false
        for _, v in ipairs(Lighting:GetChildren()) do
            if v:IsA("PostEffect") then v.Enabled = false end
        end
    end)
    makePerfBtn(3, "Otimizar Graficos", function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        workspace.CurrentCamera.FieldOfView = 70
    end)

    -- ---------- TELEPORT ----------
    local tpScroll = makeScroll()
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Teleporte para Ilhas / Gyms",
        TextColor3 = T.Accent, TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1,
        Size = UDim2.new(1,0,0,18), LayoutOrder = 1,
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, tpScroll)

    local FIXED_TP = {
        ["Mythical Gym"]    = Vector3.new(2251, 36, 1073),
        ["Jungle Gym"]      = Vector3.new(-8686, 36, 2392),
        ["Industrial Gym"]  = Vector3.new(-5563, 91, 4942),
        ["Overcharged Gym"] = Vector3.new(-3069, 183, 5003),
    }
    local function forceTP(pos)
        local hrp = Utils.getHRP()
        if not hrp then return end
        hrp.CFrame = CFrame.new(pos)
        task.wait(0.05)
        hrp.CFrame = CFrame.new(pos)
    end
    local TP_AREAS = {
        {Name="Beach",           Keyword="Beach"},
        {Name="Frost Gym",       Keyword="Frost"},
        {Name="Mythical Gym",    Keyword="Mythical"},
        {Name="Eternal Gym",     Keyword="Eternal"},
        {Name="Legends Gym",     Keyword="Legends"},
        {Name="Jungle Gym",      Keyword="Jungle"},
        {Name="Industrial Gym",  Keyword="Industrial"},
        {Name="Overcharged Gym", Keyword="Overcharged"},
        {Name="Muscle King",     Keyword="Muscle"},
        {Name="Tiny Island",     Keyword="Tiny"},
        {Name="Battle Island",   Keyword="Battle"},
        {Name="Magma Ring",      Keyword="Magma"},
        {Name="Desert Ring",     Keyword="Desert"},
        {Name="Boxing Ring",     Keyword="Boxing"},
    }
    for i, area in ipairs(TP_AREAS) do
        local card = makeCard(tpScroll, i + 2, isMobile and 36 or 52)
        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = area.Name, TextColor3 = T.Text,
            TextSize = isMobile and 10 or 12,
            BackgroundTransparency = 1, Position = UDim2.new(0,14,0,0),
            Size = UDim2.new(1,-100,1,0), TextXAlignment = Enum.TextXAlignment.Left,
            TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
        }, card)
        local goBtn = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = "Ir", TextColor3 = T.Text,
            TextSize = isMobile and 10 or 12,
            BackgroundColor3 = T.Accent, BorderSizePixel = 0,
            AutoButtonColor = false,
            AnchorPoint = Vector2.new(1,0.5), Position = UDim2.new(1,-12,0.5,0),
            Size = UDim2.new(0, isMobile and 56 or 66, 0, isMobile and 24 or 30),
            ZIndex = 6,
        }, card)
        UI.corner(goBtn, UDim.new(0, 6))
        goBtn.Activated:Connect(function()
            pcall(function()
                if FIXED_TP[area.Name] then
                    forceTP(FIXED_TP[area.Name])
                    goBtn.Text = "OK"; goBtn.TextColor3 = T.Success
                else
                    local p = AutoTP.findUp(area.Keyword)
                    if p then
                        forceTP(p.Position + Vector3.new(0,5,0))
                        goBtn.Text = "OK"; goBtn.TextColor3 = T.Success
                    else
                        goBtn.Text = "X"; goBtn.TextColor3 = T.Danger
                    end
                end
            end)
            task.wait(1)
            pcall(function()
                goBtn.Text = "Ir"; goBtn.TextColor3 = T.Text
            end)
        end)
    end

    -- ---------- BOSS ----------
    local bossScroll = makeScroll()
    local bossCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0,isMobile and 215 or 255), LayoutOrder = 1,
        ZIndex = 5,
    }, bossScroll)
    UI.corner(bossCard, UDim.new(0, 10))
    Utils.addBtnBg(bossCard, 0.15)
    local bossStroke = Utils.create("UIStroke",
        {Color = T.Boss, Thickness = 1.5, Transparency = 0.4}, bossCard)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "AUTO BOSS", TextColor3 = T.Boss,
        TextSize = isMobile and 12 or 15,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, bossCard)

    BossToggleBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = State.autoBoss.active and "DESATIVAR AUTO BOSS" or "ATIVAR AUTO BOSS",
        TextColor3 = T.Text, TextSize = isMobile and 10 or 12,
        BackgroundColor3 = State.autoBoss.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0,14,0,32), Size = UDim2.new(1,-28,0,isMobile and 26 or 34),
        ZIndex = 6,
    }, bossCard)
    UI.corner(BossToggleBtn, UDim.new(0, 8))
    local bossToggleStroke = Utils.create("UIStroke",
        {Color = State.autoBoss.active and T.Success or T.Border,
         Thickness = 1, Transparency = State.autoBoss.active and 0.1 or 0.4}, BossToggleBtn)

    BossToggleBtn.Activated:Connect(function()
        local b = State.autoBoss
        b.active = not b.active
        if b.active then
            b.mode = "idle"
            b.hasTeleported = false
            b.target, b.chestTPDone = nil, false
            b.chestCollected = false
            b.savedArenaPos = BOSS_WAIT_POS
            b.totalDamage = 0
            b.lastBossHp = 0
            
            BossToggleBtn.Text = "DESATIVAR AUTO BOSS"
            BossToggleBtn.BackgroundColor3 = T.Success
            bossToggleStroke.Color = T.Success; bossToggleStroke.Transparency = 0.1
            bossStroke.Transparency = 0.2
        else
            BossToggleBtn.Text = "ATIVAR AUTO BOSS"
            BossToggleBtn.BackgroundColor3 = T.CardDark
            bossToggleStroke.Color = T.Border; bossToggleStroke.Transparency = 0.4
            bossStroke.Transparency = 0.4
            Boss.destroyHitbox()
        end
    end)

    local hpBox = Utils.create("Frame", {
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0,
        Position = UDim2.new(0,14,0,isMobile and 64 or 72),
        Size = UDim2.new(1,-28,0, isMobile and 100 or 118),
        ZIndex = 6,
    }, bossCard)
    UI.corner(hpBox, UDim.new(0, 8))
    Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, hpBox)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "STATUS DO BOSS",
        TextColor3 = T.TextMuted, TextSize = 9, BackgroundTransparency = 1,
        Position = UDim2.new(0,12,0,6), Size = UDim2.new(1,-24,0,14),
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 7,
    }, hpBox)
    BossHpLabel = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Boss: OFF",
        TextColor3 = T.TextMuted,
        TextSize = isMobile and 11 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,12,0,24),
        Size = UDim2.new(1,-24,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 7,
    }, hpBox)
    BossHpBar = Utils.create("Frame", {
        BackgroundColor3 = T.Border, BorderSizePixel = 0,
        Position = UDim2.new(0,12,0,isMobile and 46 or 52),
        Size = UDim2.new(1,-24,0,8),
        ZIndex = 7,
    }, hpBox)
    UI.corner(BossHpBar, UDim.new(1,0))
    BossHpBarFill = Utils.create("Frame", {
        BackgroundColor3 = T.Success, BorderSizePixel = 0,
        Size = UDim2.new(0,0,1,0),
    }, BossHpBar)
    UI.corner(BossHpBarFill, UDim.new(1,0))

    State.autoBoss.damageDisplay = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack,
        Text = "DANO TOTAL: 0",
        TextColor3 = T.Fast,
        TextSize = isMobile and 12 or 14,
        BackgroundTransparency = 1,
        Position = UDim2.new(0, 12, 0, isMobile and 60 or 68),
        Size = UDim2.new(1, -24, 0, 20),
        TextXAlignment = Enum.TextXAlignment.Left,
        TextStrokeTransparency = 0.7,
        TextStrokeColor3 = Color3.fromRGB(0,0,0),
        ZIndex = 10,
    }, hpBox)

    local size10Card = makeCard(bossScroll, 2, CARD_H)
    local size10Stroke = Utils.create("UIStroke",
        {Color = AutoSize10.active and T.Success or T.Border,
         Thickness = 1.5, Transparency = AutoSize10.active and 0.1 or 0.4}, size10Card)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Size 10", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, size10Card)
    local size10Toggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = AutoSize10.active and "ON" or "OFF",
        TextColor3 = AutoSize10.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = AutoSize10.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
        ZIndex = 6,
    }, size10Card)
    UI.corner(size10Toggle, UDim.new(1,0))
    size10Toggle.Activated:Connect(function()
        AutoSize10.active = not AutoSize10.active
        if AutoSize10.active then
            size10Toggle.Text = "ON"; size10Toggle.TextColor3 = T.Text
            size10Toggle.BackgroundColor3 = T.Success
            size10Stroke.Color = T.Success; size10Stroke.Transparency = 0.1
            AutoSize10.lastApply = 0
        else
            size10Toggle.Text = "OFF"; size10Toggle.TextColor3 = T.TextMuted
            size10Toggle.BackgroundColor3 = T.CardDark
            size10Stroke.Color = T.Border; size10Stroke.Transparency = 0.4
        end
    end)

    local punchCard = makeCard(bossScroll, 3, CARD_H)
    local punchStroke = Utils.create("UIStroke",
        {Color = State.autoPunch.active and T.Success or T.Border,
         Thickness = 1.5, Transparency = State.autoPunch.active and 0.1 or 0.4}, punchCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Punch Fast", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, punchCard)
    local punchToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = State.autoPunch.active and "ON" or "OFF",
        TextColor3 = State.autoPunch.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = State.autoPunch.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
        ZIndex = 6,
    }, punchCard)
    UI.corner(punchToggle, UDim.new(1,0))
    punchToggle.Activated:Connect(function()
        State.autoPunch.active = not State.autoPunch.active
        if State.autoPunch.active then
            punchToggle.Text = "ON"; punchToggle.TextColor3 = T.Text
            punchToggle.BackgroundColor3 = T.Success
            punchStroke.Color = T.Success; punchStroke.Transparency = 0.1
        else
            punchToggle.Text = "OFF"; punchToggle.TextColor3 = T.TextMuted
            punchToggle.BackgroundColor3 = T.CardDark
            punchStroke.Color = T.Border; punchStroke.Transparency = 0.4
        end
    end)

    local farmBossCard = makeCard(bossScroll, 4, CARD_H)
    BossFarmStroke = Utils.create("UIStroke",
        {Color = State.autoBossFarm.active and T.Success or T.Border,
         Thickness = 1.5, Transparency = State.autoBossFarm.active and 0.1 or 0.4}, farmBossCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Boss e Farm", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, farmBossCard)
    BossFarmToggleBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = State.autoBossFarm.active and "ON" or "OFF",
        TextColor3 = State.autoBossFarm.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = State.autoBossFarm.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
        ZIndex = 6,
    }, farmBossCard)
    UI.corner(BossFarmToggleBtn, UDim.new(1,0))
    
    BossFarmToggleBtn.Activated:Connect(function()
        State.autoBossFarm.active = not State.autoBossFarm.active
        
        if State.autoBossFarm.active then
            BossFarmToggleBtn.Text = "ON"
            BossFarmToggleBtn.TextColor3 = T.Text
            BossFarmToggleBtn.BackgroundColor3 = T.Success
            BossFarmStroke.Color = T.Success
            BossFarmStroke.Transparency = 0.1
            
            State.autoBoss.active = true
            State.autoBoss.mode = "idle"
            State.autoBoss.hasTeleported = false
            State.autoBoss.target, State.autoBoss.chestTPDone = nil, false
            State.autoBoss.chestCollected = false
            State.autoBoss.savedArenaPos = BOSS_WAIT_POS
            State.autoBoss.totalDamage = 0
            State.autoBoss.lastBossHp = 0
            BossToggleBtn.Text = "DESATIVAR AUTO BOSS"
            BossToggleBtn.BackgroundColor3 = T.Success
            bossToggleStroke.Color = T.Success
            bossToggleStroke.Transparency = 0.1
            bossStroke.Transparency = 0.2
        else
            BossFarmToggleBtn.Text = "OFF"
            BossFarmToggleBtn.TextColor3 = T.TextMuted
            BossFarmToggleBtn.BackgroundColor3 = T.CardDark
            BossFarmStroke.Color = T.Border
            BossFarmStroke.Transparency = 0.4
            
            State.autoBoss.active = false
            BossToggleBtn.Text = "ATIVAR AUTO BOSS"
            BossToggleBtn.BackgroundColor3 = T.CardDark
            bossToggleStroke.Color = T.Border
            bossToggleStroke.Transparency = 0.4
            bossStroke.Transparency = 0.4
            Boss.destroyHitbox()
        end
    end)

    local bossOffsetCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card, BackgroundTransparency = 0.35, BorderSizePixel = 0,
        Size = UDim2.new(1,0,0, isMobile and 80 or 90), LayoutOrder = 5,
        ZIndex = 5,
    }, bossScroll)
    UI.corner(bossOffsetCard, UDim.new(0, 10))
    Utils.addBtnBg(bossOffsetCard, 0.15)
    Utils.create("UIStroke", {Color = T.Boss, Thickness = 1.5, Transparency = 0.4}, bossOffsetCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Ajuste de altura:",
        TextColor3 = T.Boss, TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, bossOffsetCard)
    local bossOffsetLabel = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack,
        Text = tostring(State.autoBoss.offsetY or 0),
        TextColor3 = T.Boss, TextSize = isMobile and 12 or 14,
        BackgroundTransparency = 1, AnchorPoint = Vector2.new(1,0),
        Position = UDim2.new(1,-14,0,8),
        Size = UDim2.new(0,60,0,18), TextXAlignment = Enum.TextXAlignment.Right,
        ZIndex = 6,
    }, bossOffsetCard)

    local bOffY = isMobile and 34 or 38
    local function makeOffsetBtn(text, xPos, bg)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = text, TextColor3 = T.Text,
            TextSize = isMobile and 11 or 12,
            BackgroundColor3 = bg, BorderSizePixel = 0, AutoButtonColor = false,
            Position = UDim2.new(xPos, 0, 0, bOffY),
            Size = UDim2.new(0.22, 0, 0, isMobile and 28 or 32),
            ZIndex = 6,
        }, bossOffsetCard)
        UI.corner(b, UDim.new(0, 6))
        return b
    end
    local bMinus5 = makeOffsetBtn("-5", 0, Color3.fromRGB(120, 50, 60))
    bMinus5.Position = UDim2.new(0, 14, 0, bOffY)
    local bMinus1 = makeOffsetBtn("-1", 0.24, Color3.fromRGB(90, 50, 55))
    bMinus1.Position = UDim2.new(0.24, 0, 0, bOffY)
    local bPlus1 = makeOffsetBtn("+1", 0, Color3.fromRGB(50, 100, 70))
    bPlus1.Position = UDim2.new(0.50, 0, 0, bOffY)
    local bPlus5 = makeOffsetBtn("+5", 0, Color3.fromRGB(50, 120, 80))
    bPlus5.Position = UDim2.new(0.76, -14, 0, bOffY)

    local function setBossOffset(val)
        State.autoBoss.offsetY = val
        bossOffsetLabel.Text = tostring(val)
    end
    bMinus5.Activated:Connect(function() setBossOffset((State.autoBoss.offsetY or 0) - 5) end)
    bMinus1.Activated:Connect(function() setBossOffset((State.autoBoss.offsetY or 0) - 1) end)
    bPlus1.Activated:Connect(function() setBossOffset((State.autoBoss.offsetY or 0) + 1) end)
    bPlus5.Activated:Connect(function() setBossOffset((State.autoBoss.offsetY or 0) + 5) end)

    -- ---------- HALLOWEEN ----------
    local halloweenScroll = makeScroll()
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold,
        Text = "HALLOWEEN BOSSES",
        TextColor3 = T.Halloween,
        TextSize = isMobile and 12 or 15,
        BackgroundTransparency = 1,
        Size = UDim2.new(1,0,0,20),
        LayoutOrder = 1,
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, halloweenScroll)

    local hwUIRefs = {}

    local function createHWBossSection(boss, order)
        local card = Utils.create("Frame", {
            BackgroundColor3 = T.Card,
            BackgroundTransparency = 0.35,
            BorderSizePixel = 0,
            Size = UDim2.new(1,0,0, isMobile and 82 or 96),
            LayoutOrder = order + 1, ZIndex = 5,
        }, halloweenScroll)
        UI.corner(card, UDim.new(0, 10))
        Utils.addBtnBg(card, 0.15)
        local cardStroke = Utils.create("UIStroke", {
            Color = boss.color, Thickness = 1.5, Transparency = 0.4,
        }, card)

        Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = boss.name,
            TextColor3 = boss.color, TextSize = isMobile and 11 or 13,
            BackgroundTransparency = 1,
            Position = UDim2.new(0,14,0,8),
            Size = UDim2.new(1,-140,0,20),
            TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
        }, card)

        Utils.create("TextLabel", {
            Font = Enum.Font.GothamMedium, Text = "Dificuldade: " .. boss.diff,
            TextColor3 = T.TextMuted, TextSize = isMobile and 9 or 10,
            BackgroundTransparency = 1,
            Position = UDim2.new(0,14,0,28),
            Size = UDim2.new(1,-140,0,14),
            TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
        }, card)

        local toggleBtn = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = "ATIVAR",
            TextColor3 = T.Text, TextSize = isMobile and 10 or 11,
            BackgroundColor3 = T.CardDark,
            BorderSizePixel = 0, AutoButtonColor = false,
            AnchorPoint = Vector2.new(1,0),
            Position = UDim2.new(1,-12,0,10),
            Size = UDim2.new(0, isMobile and 96 or 110, 0, isMobile and 28 or 32),
            ZIndex = 6,
        }, card)
        UI.corner(toggleBtn, UDim.new(1,0))

        local hpBarBg = Utils.create("Frame", {
            BackgroundColor3 = T.Border,
            BorderSizePixel = 0,
            Position = UDim2.new(0,14,0,50),
            Size = UDim2.new(1,-28,0,8),
            ZIndex = 6,
        }, card)
        UI.corner(hpBarBg, UDim.new(1,0))

        local hpFill = Utils.create("Frame", {
            BackgroundColor3 = T.Success,
            BorderSizePixel = 0,
            Size = UDim2.new(0,0,1,0),
        }, hpBarBg)
        UI.corner(hpFill, UDim.new(1,0))

        local hpText = Utils.create("TextLabel", {
            Font = Enum.Font.GothamBold, Text = "HP: 0 / 0",
            TextColor3 = T.Text, TextSize = isMobile and 9 or 10,
            BackgroundTransparency = 1,
            Position = UDim2.new(0,14,0,62),
            Size = UDim2.new(1,-28,0,16),
            TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
        }, card)

        toggleBtn.Activated:Connect(function()
            if boss.active then
                boss.active = false
                HWBoss.destroyHitbox(boss)
                boss.target = nil
                boss.mode = "idle"
            else
                HWBoss.desativarTodos(boss)
                boss.active = true
                boss.mode = "idle"
                boss.target = nil
            end
        end)

        hwUIRefs[boss.id] = {
            toggleBtn = toggleBtn, cardStroke = cardStroke,
            hpFill = hpFill, hpText = hpText,
        }
    end

    for i, boss in ipairs(HW_BOSSES) do
        createHWBossSection(boss, i + 1)
    end

    local globalOffsetCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card,
        BackgroundTransparency = 0.35,
        BorderSizePixel = 0,
        Size = UDim2.new(1,0,0, isMobile and 80 or 90),
        LayoutOrder = 101, ZIndex = 5,
    }, halloweenScroll)
    UI.corner(globalOffsetCard, UDim.new(0, 10))
    Utils.addBtnBg(globalOffsetCard, 0.15)
    Utils.create("UIStroke", {Color = T.Halloween, Thickness = 1.5, Transparency = 0.4}, globalOffsetCard)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold,
        Text = "Ajuste de altura (todos):",
        TextColor3 = T.Halloween,
        TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1,
        Position = UDim2.new(0,14,0,8),
        Size = UDim2.new(1,-28,0,18),
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 6,
    }, globalOffsetCard)

    local globalOffsetLabel = Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "0",
        TextColor3 = T.Halloween, TextSize = isMobile and 12 or 14,
        BackgroundTransparency = 1,
        AnchorPoint = Vector2.new(1,0),
        Position = UDim2.new(1,-14,0,8),
        Size = UDim2.new(0,60,0,18),
        TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 6,
    }, globalOffsetCard)

    local gOffY = isMobile and 34 or 38
    local function makeGOffsetBtn(text, xPos, bg)
        local b = Utils.create("TextButton", {
            Font = Enum.Font.GothamBold, Text = text, TextColor3 = T.Text,
            TextSize = isMobile and 11 or 12,
            BackgroundColor3 = bg, BorderSizePixel = 0, AutoButtonColor = false,
            Position = UDim2.new(xPos, 0, 0, gOffY),
            Size = UDim2.new(0.22, 0, 0, isMobile and 28 or 32),
            ZIndex = 6,
        }, globalOffsetCard)
        UI.corner(b, UDim.new(0, 6))
        return b
    end
    local gMinus5 = makeGOffsetBtn("-5", 0, Color3.fromRGB(120, 50, 60))
    gMinus5.Position = UDim2.new(0, 14, 0, gOffY)
    local gMinus1 = makeGOffsetBtn("-1", 0.24, Color3.fromRGB(90, 50, 55))
    gMinus1.Position = UDim2.new(0.24, 0, 0, gOffY)
    local gPlus1 = makeGOffsetBtn("+1", 0, Color3.fromRGB(50, 100, 70))
    gPlus1.Position = UDim2.new(0.50, 0, 0, gOffY)
    local gPlus5 = makeGOffsetBtn("+5", 0, Color3.fromRGB(50, 120, 80))
    gPlus5.Position = UDim2.new(0.76, -14, 0, gOffY)

    local function setGlobalOffset(val)
        for _, b in ipairs(HW_BOSSES) do
            b.offsetY = val
        end
        globalOffsetLabel.Text = tostring(val)
    end
    gMinus5.Activated:Connect(function() setGlobalOffset((HW_BOSSES[1].offsetY or 0) - 5) end)
    gMinus1.Activated:Connect(function() setGlobalOffset((HW_BOSSES[1].offsetY or 0) - 1) end)
    gPlus1.Activated:Connect(function() setGlobalOffset((HW_BOSSES[1].offsetY or 0) + 1) end)
    gPlus5.Activated:Connect(function() setGlobalOffset((HW_BOSSES[1].offsetY or 0) + 5) end)

    local tpHalloweenCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card,
        BackgroundTransparency = 0.35,
        BorderSizePixel = 0,
        Size = UDim2.new(1,0,0, isMobile and 54 or 62),
        LayoutOrder = 102, ZIndex = 5,
    }, halloweenScroll)
    UI.corner(tpHalloweenCard, UDim.new(0, 10))
    Utils.addBtnBg(tpHalloweenCard, 0.15)
    Utils.create("UIStroke", {Color = T.Halloween, Thickness = 1.5, Transparency = 0.4}, tpHalloweenCard)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold,
        Text = "Teleporte para Ilha Halloween",
        TextColor3 = T.Halloween,
        TextSize = isMobile and 11 or 13,
        BackgroundTransparency = 1,
        AnchorPoint = Vector2.new(0, 0.5),
        Position = UDim2.new(0, 14, 0.5, 0),
        Size = UDim2.new(1, -140, 1, 0),
        TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, tpHalloweenCard)

    local tpHalloweenBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = "IR",
        TextColor3 = T.Text,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = T.Halloween,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1, 0.5),
        Position = UDim2.new(1, -14, 0.5, 0),
        Size = UDim2.new(0, isMobile and 70 or 86, 0, isMobile and 32 or 38),
        ZIndex = 6,
    }, tpHalloweenCard)
    UI.corner(tpHalloweenBtn, UDim.new(0, 8))

    tpHalloweenBtn.Activated:Connect(function()
        tpHalloweenBtn.Text = "..."
        tpHalloweenBtn.TextColor3 = T.TextDark
        pcall(TP_HalloweenIsland, "scarecrow")
        task.wait(0.8)
        tpHalloweenBtn.Text = "OK!"
        tpHalloweenBtn.TextColor3 = T.Success
        task.wait(1.5)
        tpHalloweenBtn.Text = "IR"
        tpHalloweenBtn.TextColor3 = T.Text
    end)

    task.spawn(function()
        while MainGui and MainGui.Parent and Utils.alive() do
            pcall(function()
                for _, boss in ipairs(HW_BOSSES) do
                    local refs = hwUIRefs[boss.id]
                    if refs then
                        if boss.active then
                            refs.toggleBtn.Text = "DESATIVAR"
                            refs.toggleBtn.BackgroundColor3 = T.Success
                            refs.cardStroke.Transparency = 0.1
                        else
                            refs.toggleBtn.Text = "ATIVAR"
                            refs.toggleBtn.BackgroundColor3 = T.CardDark
                            refs.cardStroke.Transparency = 0.4
                        end

                        local pct = boss.maxHp > 0 and (boss.hp / boss.maxHp) or 0
                        refs.hpFill.Size = UDim2.new(math.clamp(pct, 0, 1), 0, 1, 0)
                        refs.hpText.Text = string.format("HP: %s / %s  (%.0f%%)",
                            Utils.formatNumber(boss.hp),
                            Utils.formatNumber(boss.maxHp),
                            pct * 100)

                        if pct > 0.6 then refs.hpFill.BackgroundColor3 = T.Success
                        elseif pct > 0.3 then refs.hpFill.BackgroundColor3 = T.Gold
                        else refs.hpFill.BackgroundColor3 = T.Danger end
                    end
                end
            end)
            task.wait(0.2)
        end
    end)

    -- ==========================================================
    -- HALLOWEEN PASS
    -- ==========================================================
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold,
        Text = "HALLOWEEN PASS",
        TextColor3 = T.Halloween,
        TextSize = isMobile and 12 or 15,
        BackgroundTransparency = 1,
        Size = UDim2.new(1,0,0,20),
        LayoutOrder = 103,
        TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5,
    }, halloweenScroll)

    local hwPassCard = Utils.create("Frame", {
        BackgroundColor3 = T.Card,
        BackgroundTransparency = 0.35,
        BorderSizePixel = 0,
        Size = UDim2.new(1,0,0, isMobile and 136 or 158),
        LayoutOrder = 104, ZIndex = 5,
    }, halloweenScroll)
    UI.corner(hwPassCard, UDim.new(0, 10))
    Utils.addBtnBg(hwPassCard, 0.15)
    Utils.create("UIStroke", {Color = T.Halloween, Thickness = 1.5, Transparency = 0.4}, hwPassCard)

    local qToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "AUTO CLAIM QUEST (TODAS)",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12,
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, 12),
        Size = UDim2.new(1, -28, 0, isMobile and 30 or 36),
        ZIndex = 6,
    }, hwPassCard)
    UI.corner(qToggle, UDim.new(1,0))

    local bToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "AUTO BYPASS (TIERS 1-10)",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12,
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, isMobile and 48 or 56),
        Size = UDim2.new(1, -28, 0, isMobile and 30 or 36),
        ZIndex = 6,
    }, hwPassCard)
    UI.corner(bToggle, UDim.new(1,0))

    local prestigeBtn = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold, Text = "AUTO PRESTIGE",
        TextColor3 = T.TextMuted, TextSize = isMobile and 10 or 12,
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, isMobile and 84 or 100),
        Size = UDim2.new(1, -28, 0, isMobile and 30 or 36),
        ZIndex = 6,
    }, hwPassCard)
    UI.corner(prestigeBtn, UDim.new(1,0))

    qToggle.Activated:Connect(function()
        AutoClaimQuest.active = not AutoClaimQuest.active
        if AutoClaimQuest.active then
            qToggle.Text = "AUTO CLAIM QUEST: ON"
            qToggle.TextColor3 = T.Text
            qToggle.BackgroundColor3 = T.Success
        else
            qToggle.Text = "AUTO CLAIM QUEST (TODAS)"
            qToggle.TextColor3 = T.TextMuted
            qToggle.BackgroundColor3 = T.CardDark
        end
    end)

    bToggle.Activated:Connect(function()
        AutoBypass.active = not AutoBypass.active
        if AutoBypass.active then
            bToggle.Text = "AUTO BYPASS: ON"
            bToggle.TextColor3 = T.Text
            bToggle.BackgroundColor3 = T.Success
        else
            bToggle.Text = "AUTO BYPASS (TIERS 1-10)"
            bToggle.TextColor3 = T.TextMuted
            bToggle.BackgroundColor3 = T.CardDark
        end
    end)

    prestigeBtn.Activated:Connect(function()
        AutoPrestige.active = not AutoPrestige.active
        if AutoPrestige.active then
            prestigeBtn.Text = "AUTO PRESTIGE: ON"
            prestigeBtn.TextColor3 = T.Text
            prestigeBtn.BackgroundColor3 = T.Success
        else
            prestigeBtn.Text = "AUTO PRESTIGE"
            prestigeBtn.TextColor3 = T.TextMuted
            prestigeBtn.BackgroundColor3 = T.CardDark
        end
    end)

    -- ---------- GIFT ----------
    local giftScroll = makeScroll()
    local giftCard = makeCard(giftScroll, 2, CARD_H)
    Utils.create("UIStroke",
        {Color = AutoClaimGift.active and T.Success or T.Border,
         Thickness = 1.5, Transparency = AutoClaimGift.active and 0.1 or 0.4}, giftCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Claim Gift", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, giftCard)
    local giftToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = AutoClaimGift.active and "ON" or "OFF",
        TextColor3 = AutoClaimGift.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = AutoClaimGift.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
        ZIndex = 6,
    }, giftCard)
    UI.corner(giftToggle, UDim.new(1,0))
    giftToggle.Activated:Connect(function()
        AutoClaimGift.active = not AutoClaimGift.active
        if AutoClaimGift.active then
            AutoClaimGift.proximoGift = 1
            giftToggle.Text = "ON"; giftToggle.TextColor3 = T.Text
            giftToggle.BackgroundColor3 = T.Success
        else
            giftToggle.Text = "OFF"; giftToggle.TextColor3 = T.TextMuted
            giftToggle.BackgroundColor3 = T.CardDark
        end
    end)

    -- ---------- FORTUNE ----------
    local fortuneScroll = makeScroll()
    local fortuneCard = makeCard(fortuneScroll, 2, CARD_H)
    Utils.create("UIStroke",
        {Color = AutoFortune.active and T.Success or T.Border,
         Thickness = 1.5, Transparency = AutoFortune.active and 0.1 or 0.4}, fortuneCard)
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBold, Text = "Auto Spin Wheel", TextColor3 = T.Text,
        TextSize = isMobile and 11 or 14, BackgroundTransparency = 1, AnchorPoint = Vector2.new(0,0.5),
        Position = UDim2.new(0,14,0.5,0),
        Size = UDim2.new(1,-130,1,0), TextXAlignment = Enum.TextXAlignment.Left,
        TextYAlignment = Enum.TextYAlignment.Center, ZIndex = 6,
    }, fortuneCard)
    local fortuneToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = AutoFortune.active and "ON" or "OFF",
        TextColor3 = AutoFortune.active and T.Text or T.TextMuted,
        TextSize = isMobile and 11 or 13,
        BackgroundColor3 = AutoFortune.active and T.Success or T.CardDark,
        BorderSizePixel = 0, AutoButtonColor = false,
        AnchorPoint = Vector2.new(1,0.5),
        Position = UDim2.new(1,-14,0.5,0),
        Size = UDim2.new(0, isMobile and 80 or 100, 0, isMobile and 28 or 36),
        ZIndex = 6,
    }, fortuneCard)
    UI.corner(fortuneToggle, UDim.new(1,0))
    fortuneToggle.Activated:Connect(function()
        AutoFortune.active = not AutoFortune.active
        if AutoFortune.active then
            AutoFortune.lastSpin = 0
            fortuneToggle.Text = "ON"; fortuneToggle.TextColor3 = T.Text
            fortuneToggle.BackgroundColor3 = T.Success
        else
            fortuneToggle.Text = "OFF"; fortuneToggle.TextColor3 = T.TextMuted
            fortuneToggle.BackgroundColor3 = T.CardDark
        end
    end)

    -- ==========================================================
    -- ABA STORE (STRONG BLOXIAN STORE)
    -- ==========================================================
    local storeScroll = makeScroll()

    local shopCard = makeCard(storeScroll, 1, isMobile and 130 or 160)
    Utils.create("UIStroke", {Color = T.Success, Thickness = 1.5, Transparency = 0.4}, shopCard)
    
    Utils.create("TextLabel", {
        Font = Enum.Font.GothamBlack, Text = "STRONG BLOXIAN STORE", TextColor3 = T.Success,
        TextSize = isMobile and 14 or 18, BackgroundTransparency = 1, Position = UDim2.new(0,14,0,10),
        Size = UDim2.new(1,-28,0,24), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, shopCard)

    Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, 
        Text = "Teleporta para workspace.MedalShop e compra\nautomaticamente Pets em estoque se tiver MedalCoin.", 
        TextColor3 = T.TextMuted,
        TextSize = isMobile and 10 or 12, BackgroundTransparency = 1, Position = UDim2.new(0,14,0,36),
        Size = UDim2.new(1,-28,0,40), TextXAlignment = Enum.TextXAlignment.Left,
        TextWrapped = true, ZIndex = 6,
    }, shopCard)

    local shopToggle = Utils.create("TextButton", {
        Font = Enum.Font.GothamBold,
        Text = "ATIVAR AUTO COMPRA",
        TextColor3 = T.TextMuted, TextSize = isMobile and 11 or 13,
        BackgroundColor3 = T.CardDark, BorderSizePixel = 0, AutoButtonColor = false,
        Position = UDim2.new(0, 14, 0, isMobile and 84 or 94),
        Size = UDim2.new(1, -28, 0, isMobile and 32 or 38),
        ZIndex = 6,
    }, shopCard)
    UI.corner(shopToggle, UDim.new(0, 8))
    local shopStroke = Utils.create("UIStroke", {Color = T.Border, Thickness = 1, Transparency = 0.4}, shopToggle)

    local shopStatus = Utils.create("TextLabel", {
        Font = Enum.Font.GothamMedium, Text = "Status: Aguardando...",
        TextColor3 = T.TextDark, TextSize = isMobile and 10 or 12,
        BackgroundTransparency = 1, Position = UDim2.new(0,14,0, isMobile and 120 or 136),
        Size = UDim2.new(1,-28,0,18), TextXAlignment = Enum.TextXAlignment.Left,
        ZIndex = 6,
    }, shopCard)

    shopToggle.Activated:Connect(function()
        State.autoMedalShop.active = not State.autoMedalShop.active
        if State.autoMedalShop.active then
            shopToggle.Text = "DESATIVAR AUTO COMPRA"
            shopToggle.TextColor3 = T.Text
            shopToggle.BackgroundColor3 = T.Success
            shopStroke.Color = T.Success; shopStroke.Transparency = 0.1
            shopStatus.Text = "Status: Verificando estoque de Pets..."
            shopStatus.TextColor3 = T.Welcome
        else
            shopToggle.Text = "ATIVAR AUTO COMPRA"
            shopToggle.TextColor3 = T.TextMuted
            shopToggle.BackgroundColor3 = T.CardDark
            shopStroke.Color = T.Border; shopStroke.Transparency = 0.4
            shopStatus.Text = "Status: Desativado"
            shopStatus.TextColor3 = T.TextDark
        end
    end)

    -- ==========================================================
    -- LOOP DA LOJA (MedalCoin + só Pets em estoque + saldo)
    -- ==========================================================
    task.spawn(function()
        local PET_KEYWORDS = {
            "pet","egg","zeus","ares","poseidon","hades","athena","apollo",
            "hermes","artemis","thunder","ironfang","tide","spark",
            "stormrunner","wrath","colossus","titan","dragon","phoenix",
            "griffin","hydra","cerberus","minotaur","cyclops","kraken",
        }

        local MEDAL_NAMES = {
            "MedalCoin", "MedalCoins", "MedalCoinValue",
            "Medals", "Medal", "OlympicMedals", "Medalhas",
        }

        local function safe(fn, ...)
            local ok, a, b = pcall(fn, ...)
            if ok then return a, b end
            return nil, tostring(a)
        end

        local function getMedalRemote()
            local re = ReplicatedStorage:FindFirstChild("rEvents")
            local r = re and re:FindFirstChild("medalShopRemote")
            if r and r:IsA("RemoteFunction") then return r end
            return nil
        end

        local function getMedalOpen()
            local re = ReplicatedStorage:FindFirstChild("rEvents")
            local r = re and re:FindFirstChild("medalShopOpen")
            if r and r:IsA("RemoteEvent") then return r end
            return nil
        end

        local function getMedalShopPos()
            local m = workspace:FindFirstChild("MedalShop")
            if not m then
                for _, obj in ipairs(workspace:GetDescendants()) do
                    if obj.Name == "MedalShop" and (obj:IsA("Model") or obj:IsA("BasePart")) then
                        m = obj
                        break
                    end
                end
            end
            if not m then return nil end

            if m:IsA("BasePart") then
                return m.Position
            end

            if m:IsA("Model") then
                local pp = m.PrimaryPart
                if pp then return pp.Position end
                for _, d in ipairs(m:GetDescendants()) do
                    if d:IsA("BasePart") then return d.Position end
                end
            end

            return nil
        end

        -- Lê MedalCoin (prioridade) e fallbacks
        local function getPlayerMedalCoins()
            local best = 0

            local stats = Player:FindFirstChild("leaderstats")
            if stats then
                for _, name in ipairs(MEDAL_NAMES) do
                    local v = stats:FindFirstChild(name)
                    if v and v:IsA("ValueBase") then
                        local n = tonumber(v.Value) or 0
                        if n > best then best = n end
                    end
                end
            end

            for _, contName in ipairs({"Stats","PlayerStats","OlympicStats","Data"}) do
                local c = Player:FindFirstChild(contName)
                if c then
                    for _, name in ipairs(MEDAL_NAMES) do
                        local v = c:FindFirstChild(name)
                        if v and v:IsA("ValueBase") then
                            local n = tonumber(v.Value) or 0
                            if n > best then best = n end
                        end
                    end
                end
            end

            for _, name in ipairs(MEDAL_NAMES) do
                local ok, val = pcall(function() return Player:GetAttribute(name) end)
                if ok and type(val) == "number" and val > best then best = val end
            end

            return best
        end

        local function isPetOffer(offer)
            local key = Utils.lower(offer.Key or "")
            local nm  = Utils.lower(offer.ItemName or "")
            for _, kw in ipairs(PET_KEYWORDS) do
                if key:find(kw, 1, true) or nm:find(kw, 1, true) then return true end
            end
            return false
        end

        local function teleportToShop()
            local pos = getMedalShopPos()
            if not pos then return false end
            local hrp = Utils.getHRP()
            if not hrp then return false end
            safe(function()
                hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
                hrp.Velocity = Vector3.zero
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.RotVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
            end)
            task.wait(0.4)
            safe(function() hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0)) end)
            task.wait(0.5)
            return true
        end

        local function safeShowMedalShopGui()
            safe(function()
                local pg = Player:FindFirstChild("PlayerGui")
                local gameGui = pg and pg:FindFirstChild("gameGui")
                local medalShopUI = gameGui and gameGui:FindFirstChild("MedalShop")
                if medalShopUI and medalShopUI:IsA("GuiObject") then
                    if not medalShopUI.Visible then
                        medalShopUI.Visible = true
                    end
                end
            end)
        end

        local lastSignature  = ""
        local hasTeleported  = false
        local lastBuyAttempt = 0
        local lastGetState   = 0
        local GetStateCooldown = 4

        while Utils.alive() do
            if State.autoMedalShop.active then
                local now = tick()
                if now - lastGetState >= GetStateCooldown then
                    lastGetState = now
                    local remote = getMedalRemote()
                    if not remote then
                        safe(function()
                            shopStatus.Text = "Status: medalShopRemote não encontrado."
                            shopStatus.TextColor3 = T.Danger
                        end)
                    else
                        local ok, state = pcall(function()
                            return remote:InvokeServer("GetState")
                        end)

                        if not ok or type(state) ~= "table" or state.Ok == false then
                            safe(function()
                                shopStatus.Text = "Status: Fora da loja ou erro ao ler."
                                shopStatus.TextColor3 = T.Danger
                            end)
                            if not hasTeleported then
                                hasTeleported = true
                                if teleportToShop() then
                                    safeShowMedalShopGui()
                                end
                            end
                        else
                            local offers = state.Offers
                            if type(offers) ~= "table" or #offers == 0 then
                                safe(function()
                                    shopStatus.Text = "Status: Loja vazia ou sem ofertas."
                                    shopStatus.TextColor3 = T.TextMuted
                                end)
                            else
                                local sigParts = {}
                                for _, o in ipairs(offers) do
                                    if type(o) == "table" then
                                        table.insert(sigParts, string.format("%s:%s:%d:%s",
                                            tostring(o.SlotIndex), tostring(o.Key),
                                            tonumber(o.Remaining) or 0, tostring(o.InStock)))
                                    end
                                end
                                local signature = table.concat(sigParts, "|")
                                if signature ~= lastSignature then
                                    lastSignature = signature
                                    hasTeleported = false
                                end

                                local coins = getPlayerMedalCoins()

                                local petsInStock = {}
                                for _, o in ipairs(offers) do
                                    if type(o) == "table"
                                       and isPetOffer(o)
                                       and (tonumber(o.Remaining) or 0) > 0
                                       and o.InStock ~= false then
                                        table.insert(petsInStock, o)
                                    end
                                end

                                if #petsInStock == 0 then
                                    safe(function()
                                        shopStatus.Text = "Status: Nenhum Pet em estoque. Monitorando..."
                                        shopStatus.TextColor3 = T.TextMuted
                                    end)
                                else
                                    local affordable = nil
                                    local cheapestNoBudget = nil
                                    for _, o in ipairs(petsInStock) do
                                        local price = tonumber(o.Price) or math.huge
                                        if coins >= price then
                                            if not affordable
                                               or price > (tonumber(affordable.Price) or 0) then
                                                affordable = o
                                            end
                                        else
                                            if not cheapestNoBudget
                                               or price < (tonumber(cheapestNoBudget.Price) or math.huge) then
                                                cheapestNoBudget = o
                                            end
                                        end
                                    end

                                    if not affordable then
                                        safe(function()
                                            local need = cheapestNoBudget and cheapestNoBudget.Price or 0
                                            shopStatus.Text = string.format(
                                                "Status: Pet em estoque, mas faltam MedalCoins (precisa %s / tem %s).",
                                                Utils.formatNumber(need), Utils.formatNumber(coins))
                                            shopStatus.TextColor3 = T.Danger
                                        end)
                                    elseif (tick() - lastBuyAttempt) >= 2 then
                                        lastBuyAttempt = tick()

                                        if not hasTeleported then
                                            hasTeleported = true
                                            safe(function()
                                                shopStatus.Text = string.format(
                                                    "Status: Pet '%s' (x%d) — teleportando para comprar...",
                                                    tostring(affordable.ItemName),
                                                    tonumber(affordable.Remaining) or 0)
                                                shopStatus.TextColor3 = T.Welcome
                                            end)
                                            teleportToShop()
                                            safeShowMedalShopGui()

                                            local openRemote = getMedalOpen()
                                            if openRemote then
                                                local opened = false
                                                local conn
                                                safe(function()
                                                    conn = openRemote.OnClientEvent:Connect(function() opened = true end)
                                                end)
                                                task.wait(2)
                                                safe(function() if conn then conn:Disconnect() end end)
                                            else
                                                task.wait(1)
                                            end
                                            safeShowMedalShopGui()
                                            task.wait(0.5)
                                        end

                                        safe(function()
                                            shopStatus.Text = string.format(
                                                "Status: Comprando '%s' (x%d) por %s MedalCoins...",
                                                tostring(affordable.ItemName),
                                                tonumber(affordable.Remaining) or 0,
                                                Utils.formatNumber(affordable.Price))
                                            shopStatus.TextColor3 = T.Success
                                        end)

                                        local buyOk, buyResult = pcall(function()
                                            return remote:InvokeServer("Purchase", {
                                                SlotIndex = affordable.SlotIndex,
                                                Key       = affordable.Key,
                                            })
                                        end)

                                        if buyOk and type(buyResult) == "table" and buyResult.Ok == true then
                                            safe(function()
                                                shopStatus.Text = "Status: Pet comprado! Monitorando..."
                                                shopStatus.TextColor3 = T.Success
                                            end)
                                        else
                                            local reason = (buyOk and type(buyResult) == "table" and buyResult.Reason) or "?"
                                            local reasonMap = {
                                                notEnoughMedals = "MedalCoins insuficientes",
                                                outOfStock      = "Sem estoque",
                                                stale           = "Loja restocou (re-tentar)",
                                                notInGym        = "Fora da loja (re-teleportando)",
                                            }
                                            safe(function()
                                                shopStatus.Text = "Status: Falha (" .. (reasonMap[reason] or tostring(reason)) .. ")."
                                                shopStatus.TextColor3 = T.Danger
                                            end)

                                            if reason == "notInGym" or reason == "stale" then
                                                hasTeleported = false
                                            end
                                        end

                                        task.wait(1.5)
                                    end
                                end
                            end
                        end
                    end
                end
            end
            task.wait(1)
        end
    end)

    local TAB_MAP = {
        Main = mainScroll, Farm = farmScroll,
        Rebirth = rebirthScroll, Config = settingsScroll,
        Stats = statsScroll, Teleport = tpScroll,
        Boss = bossScroll, Halloween = halloweenScroll,
        Gift = giftScroll, Fortune = fortuneScroll,
        Store = storeScroll,
    }

    local currentTabIndex = 1
    local function setTab(name)
        for key, info in pairs(tabInfo) do
            local active = (key == name)
            info.btn.TextColor3 = active and T.Text or T.TextDark
            if TAB_MAP[key] then TAB_MAP[key].Visible = active end
        end
        for i, n in ipairs(tabOrder) do
            if n == name then
                currentTabIndex = i
                break
            end
        end
        if tabIndicator then
            tabIndicator.Position = UDim2.new(0, 4 + (currentTabIndex-1)*(TAB_W+TAB_GAP), 1, -3)
            tabIndicator.Size = UDim2.new(0, TAB_W, 0, 2)
        end
        scrollTabsTo(currentTabIndex)
    end

    for key, info in pairs(tabInfo) do
        info.btn.Activated:Connect(function() setTab(key) end)
    end

    setTab("Main")

    task.spawn(function()
        local hue = 0
        while MainGui and MainGui.Parent and Utils.alive() do
            if State.rgbMode then
                pcall(function()
                    hue = (hue + 0.005) % 1
                    local newColor = Color3.fromHSV(hue, 0.75, 0.9)
                    
                    T.Accent = newColor
                    mainStroke.Color = newColor
                    tabIndicator.BackgroundColor3 = newColor
                    scrollThumb.BackgroundColor3 = newColor
                    
                    for _, l in ipairs(accentLabels) do 
                        l.TextColor3 = newColor 
                    end
                end)
            end
            task.wait(0.05)
        end
    end)

    task.defer(function()
        local cam = workspace.CurrentCamera
        if not cam then return end
        local vp = cam.ViewportSize
        local curW = container.AbsoluteSize.X
        local curH = container.AbsoluteSize.Y
        if curW > vp.X * 0.92 or curH > vp.Y * 0.65 then
            container.Size = UDim2.new(0, math.min(curW, vp.X * 0.92),
                                        0, math.min(curH, vp.Y * 0.65))
        end
    end)

    task.delay(0.05, function()
        if not (container and container.Parent) then return end
        TweenService:Create(container,
            TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out),
            {Size = FULL}):Play()
        TweenService:Create(container,
            TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
            {BackgroundTransparency = 0}):Play()
        TweenService:Create(mainStroke,
            TweenInfo.new(0.4), {Transparency = 0.2}):Play()
    end)
end

--==========================================================
-- BOOT
--==========================================================
openLauncher()

local lastVp = Vector2.new()
task.spawn(function()
    while Utils.alive() do
        pcall(function()
            local cam = workspace.CurrentCamera
            if cam then
                local vp = cam.ViewportSize
                if vp.X ~= lastVp.X or vp.Y ~= lastVp.Y then
                    lastVp = vp
                    for _, gui in ipairs({MainGui, AutoFastGui, FloatingBtnGui}) do
                        if gui and gui.Parent then
                            for _, d in ipairs(gui:GetDescendants()) do
                                if d:IsA("ScrollingFrame") then
                                    local cs = d.CanvasSize
                                    d.CanvasSize = UDim2.new(0,0,0, cs.Y.Offset)
                                    task.wait()
                                    d.CanvasSize = cs
                                end
                            end
                        end
                    end
                end
            end
        end)
        task.wait(0.5)
    end
end)

print("[Private Script v18.12.15] Carregado — Bem-vindo, " .. Player.Name .. "!")
