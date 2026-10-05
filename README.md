--!nocheck
--!nolint

if getgenv().UN_ADDON and getgenv().UN_ADDON.loaded then
    pcall(function() getgenv().UN_ADDON.unload() end)
end
local ADDON = { loaded = true }
getgenv().UN_ADDON = ADDON

local Players    = game:GetService("Players")
local RS         = game:GetService("RunService")
local Replicated = game:GetService("ReplicatedStorage")
local Stats      = game:GetService("Stats")
local UIS        = game:GetService("UserInputService")
local Tween      = game:GetService("TweenService")
local Debris     = game:GetService("Debris")
local HS         = game:GetService("HttpService")
local L          = game:GetService("Lighting")
local LP         = Players.LocalPlayer
if not LP then
    Players:GetPropertyChangedSignal("LocalPlayer"):Wait()
    LP = Players.LocalPlayer
end
local Cam        = workspace.CurrentCamera

-- ============================================================
-- NOTIFICATIONS
-- ============================================================
local PlayerGuiForToast = LP:WaitForChild("PlayerGui")
local toastGui = PlayerGuiForToast:FindFirstChild("UN_ADDON_TOASTS")
if not toastGui then
    toastGui = Instance.new("ScreenGui")
    toastGui.Name = "UN_ADDON_TOASTS"
    toastGui.ResetOnSpawn = false
    toastGui.DisplayOrder = 999998
    toastGui.Parent = PlayerGuiForToast
end

local toastIndex = 0
local function fallbackNotif(text, on)
    toastIndex = toastIndex + 1
    local idx = toastIndex
    local toast = Instance.new("TextLabel")
    toast.Size = UDim2.new(0, 230, 0, 30)
    toast.Position = UDim2.new(1, -250, 0, 100 + (idx % 8) * 34)
    toast.BackgroundColor3 = on and Color3.fromRGB(40,160,70) or Color3.fromRGB(80,25,25)
    toast.TextColor3 = Color3.new(1,1,1)
    toast.Text = "  " .. tostring(text) .. "  " .. (on and "[ON]" or "[OFF]")
    toast.Font = Enum.Font.GothamBold
    toast.TextSize = 12
    toast.BorderSizePixel = 0
    toast.TextXAlignment = Enum.TextXAlignment.Left
    toast.BackgroundTransparency = 0.15
    toast.Parent = toastGui
    local c = Instance.new("UICorner", toast); c.CornerRadius = UDim.new(0, 6)
    local s = Instance.new("UIStroke", toast)
    s.Color = on and Color3.fromRGB(120,255,160) or Color3.fromRGB(255,140,140)
    s.Thickness = 1.2
    task.delay(2.2, function()
        if toast.Parent then
            Tween:Create(toast, TweenInfo.new(0.35), {BackgroundTransparency=1, TextTransparency=1}):Play()
            task.wait(0.4)
            if toast.Parent then toast:Destroy() end
        end
    end)
end

local function notif(text, on)
    if getgenv().UN and type(getgenv().UN.notif) == "function" then
        local ok = pcall(function() getgenv().UN.notif(text, on) end)
        if ok then return end
    end
    pcall(fallbackNotif, text, on)
end

-- ============================================================
-- BIND SYSTEM
-- ============================================================
local BINDS = {}
local BIND_FILE = "un_addon_binds.json"
local bindPicking = nil

local function inputName(input)
    if not input then return "NONE" end
    if typeof(input) == "EnumItem" then
        return input.Name
    end
    return tostring(input)
end

local function parseInput(str)
    if type(str) ~= "string" or str == "" then return nil end
    return Enum.KeyCode[str] or Enum.UserInputType[str]
end

local function saveBinds()
    local out = {}
    for k, v in pairs(BINDS) do
        if type(v)=="table" and v.input then
            out[k] = { input = inputName(v.input), name = v.name }
        end
    end
    pcall(function()
        if writefile then writefile(BIND_FILE, HS:JSONEncode(out)) end
    end)
end

local function loadBinds()
    pcall(function()
        if readfile and isfile and isfile(BIND_FILE) then
            local data = HS:JSONDecode(readfile(BIND_FILE))
            for k, v in pairs(data) do
                if BINDS[k] and v.input then
                    BINDS[k].input = parseInput(v.input)
                end
            end
        end
    end)
end

local function registerBind(key, name, getter, setter)
    BINDS[key] = { input = nil, name = name, getter = getter, setter = setter }
end

local function openBindPicker(key, name)
    bindPicking = { key = key, name = name, at = os.clock() }
    BINDS._ignoreMouseUntil = os.clock() + 0.6
    local lg = getgenv().UN_LANG
    local tail = lg=="ru" and " (Р¶РјРё РєР»Р°РІРёС€Сѓ, ESC=РѕС‚РјРµРЅР°)" or lg=="uk" and " (РЅР°С‚РёСЃРЅРё РєР»Р°РІС–С€Сѓ, ESC=СЃРєР°СЃСѓРІР°С‚Рё)" or " (press key, ESC=cancel)"
    notif("bind "..name..tail, true)
end

-- ============================================================
-- BOOT: device + language (2 panels before main script)
-- ============================================================
getgenv().UN_DEVICE = nil
getgenv().UN_LANG = nil
getgenv().UN_BOOT_OK = false
do
local function bootTrace(s)
    pcall(function() if writefile then writefile("un_boot_error.txt", "stage:"..tostring(s)) end end)
end
local function bootShowInner()
    local pg = LP:WaitForChild("PlayerGui")
    pcall(function() local o=pg:FindFirstChild("UN_BOOT") if o then o:Destroy() end end)
    local sg = Instance.new("ScreenGui") sg.Name="UN_BOOT" sg.ResetOnSpawn=false sg.DisplayOrder=5000000 sg.IgnoreGuiInset=true sg.Parent=pg
    local f=Instance.new("Frame",sg) f.Size=UDim2.new(0,300,0,230) f.Position=UDim2.new(0.5,-150,0.5,-115)
    f.BackgroundColor3=Color3.fromRGB(8,13,24) f.BorderSizePixel=0 f.Active=true f.Draggable=true
    Instance.new("UICorner", f).CornerRadius = UDim.new(0,16)
    local fst = Instance.new("UIStroke", f); fst.Color=Color3.fromRGB(0,220,255); fst.Thickness=2; fst.Transparency=0.25
    local bscale=Instance.new("UIScale",f); bscale.Scale=0.9
    pcall(function() game:GetService("TweenService"):Create(bscale, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale=1}):Play() end)
    local title=Instance.new("TextLabel",f) title.Size=UDim2.new(1,0,0,40) title.BackgroundColor3=Color3.fromRGB(10,22,36) title.BorderSizePixel=0
    Instance.new("UICorner", title).CornerRadius = UDim.new(0,16)
    title.Font=Enum.Font.GothamBlack title.TextSize=15 title.TextColor3=Color3.fromRGB(0,220,255)
    local function btn(txt,y,cb)
        getgenv().UN_BI = (getgenv().UN_BI or 0) + 1
        local my = getgenv().UN_BI
        local b=Instance.new("TextButton",f) b.Size=UDim2.new(1,-16,0,46) b.Position=UDim2.new(0,8,0,y)
        b.BackgroundColor3=Color3.fromRGB(20,34,54) b.Text=txt b.TextColor3=Color3.new(1,1,1)
        b.Font=Enum.Font.GothamBold b.TextSize=16 b.BorderSizePixel=0
        b.BackgroundTransparency=1; b.TextTransparency=1
        Instance.new("UICorner", b).CornerRadius = UDim.new(0,12)
        local bst = Instance.new("UIStroke", b); bst.Color=Color3.fromRGB(0,220,255); bst.Thickness=1; bst.Transparency=1
        task.delay(0.09*my, function()
            pcall(function()
                local tw = game:GetService("TweenService")
                tw:Create(b, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency=0, TextTransparency=0}):Play()
                tw:Create(bst, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Transparency=0.6}):Play()
            end)
        end)
        b.MouseButton1Down:Connect(function() pcall(function() b.BackgroundTransparency=0.35 end) end)
        b.MouseButton1Up:Connect(function() pcall(function() b.BackgroundTransparency=0 end) end)
        b.MouseLeave:Connect(function() pcall(function() b.BackgroundTransparency=0 end) end)
        b.MouseButton1Click:Connect(function() pcall(cb,b) end)
        return b
    end
    bootTrace("gui-built")
    title.Text="в…  STEP 1 / 2 : DEVICE"
    local b1=btn("PHONE",48,function() getgenv().UN_DEVICE="Mobile" end)
    local b2=btn("PC",102,function() getgenv().UN_DEVICE="PC" end)
    local hint=Instance.new("TextLabel",f) hint.Size=UDim2.new(1,0,0,20) hint.Position=UDim2.new(0,0,1,-22)
    hint.BackgroundTransparency=1 hint.Text="StarWare setup" hint.TextColor3=Color3.fromRGB(140,160,190)
    hint.Font=Enum.Font.GothamMedium hint.TextSize=11
    while not getgenv().UN_DEVICE do task.wait(0.1) end
    b1:Destroy() b2:Destroy() hint:Destroy()
    title.Text="в…  STEP 2 / 2 : LANGUAGE"
    f.Size=UDim2.new(0,300,0,278) f.Position=UDim2.new(0.5,-150,0.5,-139)
    btn("English",48,function() getgenv().UN_LANG="en" end)
    btn("Russian",102,function() getgenv().UN_LANG="ru" end)
    btn("Ukrainian",156,function() getgenv().UN_LANG="uk" end)
    local hint2=Instance.new("TextLabel",f) hint2.Size=UDim2.new(1,0,0,20) hint2.Position=UDim2.new(0,0,1,-22)
    hint2.BackgroundTransparency=1 hint2.Text="applies to everything" hint2.TextColor3=Color3.fromRGB(140,160,190)
    hint2.Font=Enum.Font.GothamMedium hint2.TextSize=11
    while not getgenv().UN_LANG do task.wait(0.1) end
    pcall(function() sg:Destroy() end)
    getgenv().UN_BOOT_OK=true
end
local function bootShow()
    bootTrace("start")
    local ok, err = pcall(bootShowInner)
    if not ok then
        warn("[UN BOOT] "..tostring(err))
        pcall(function() if writefile then writefile("un_boot_error.txt", tostring(err)) end end)
        getgenv().UN_DEVICE = getgenv().UN_DEVICE or "PC"
        getgenv().UN_LANG = getgenv().UN_LANG or "ru"
        pcall(function()
            local pg = LP:FindFirstChild("PlayerGui")
            if pg then local o = pg:FindFirstChild("UN_BOOT") if o then o:Destroy() end end
        end)
        getgenv().UN_BOOT_OK = true
    end
end
task.spawn(bootShow)
end

-- ============================================================
-- STATE
-- ============================================================
local SA    = { enabled = false, predict = true, walls = true }
local KS    = { enabled = false }
local KA    = { on = false, dist = 30, last = 0 }
local TRACE = { on = false, col = Color3.fromRGB(133,220,255), dur = 1, conn = nil }
local BT    = { on = false, model = nil, cache = {}, ping = 0.15, hist = {}, cap = 256, first = 1, n = 0, lastping = 0 }

local autograb_on = false
local grab_round_mod = nil
local grab_desc_conn = nil

local MOVE  = { speedOn=false, speed=50, jumpOn=false, jump=50, hipOn=false, hip=2,
                infJump=false, noclip=false, fly=false, flySpeed=60, fg=nil, fv=nil,
                gravOn=false, grav=196.2, spin=false, spinSpeed=720 }
local moveConns = { infJump=nil, noclip=nil, nc={} }
local flyConn, spinConn

local TROLL = { flinging=false, sitSpam=false, follow=false, followTarget=nil, sitLast=0 }

local fl = { trail=false, parts=false, hat=false, foot=false, rainbow=false,
             esp=false, espGhost=true, tracer=false, atmo=false, jump=false,
             bounce=false, lightning=false, orbAura=false, glow=false,
             snow=false, ghost=false, skyDust=false, bloom=false, fb=false,
             sky=false, _fog=false, antifling=false }

local TH = {
    Purple = {A=Color3.fromRGB(160,80,255),A2=Color3.fromRGB(220,160,255),BG=Color3.fromRGB(16,10,26),BG2=Color3.fromRGB(24,16,38),BG3=Color3.fromRGB(20,14,32),ON=Color3.fromRGB(40,160,70),OFF=Color3.fromRGB(80,25,25)},
    Cyan   = {A=Color3.fromRGB(60,200,255),A2=Color3.fromRGB(140,230,255),BG=Color3.fromRGB(8,18,26),BG2=Color3.fromRGB(14,28,38),BG3=Color3.fromRGB(12,22,32),ON=Color3.fromRGB(40,160,180),OFF=Color3.fromRGB(30,60,80)},
    Pink   = {A=Color3.fromRGB(255,100,200),A2=Color3.fromRGB(255,170,230),BG=Color3.fromRGB(26,10,20),BG2=Color3.fromRGB(38,16,30),BG3=Color3.fromRGB(30,14,26),ON=Color3.fromRGB(180,50,120),OFF=Color3.fromRGB(80,30,50)},
    Red    = {A=Color3.fromRGB(255,80,80),A2=Color3.fromRGB(255,160,160),BG=Color3.fromRGB(24,10,10),BG2=Color3.fromRGB(36,16,16),BG3=Color3.fromRGB(28,14,14),ON=Color3.fromRGB(180,50,50),OFF=Color3.fromRGB(80,30,30)},
    Gold   = {A=Color3.fromRGB(255,200,60),A2=Color3.fromRGB(255,230,150),BG=Color3.fromRGB(24,18,8),BG2=Color3.fromRGB(38,28,14),BG3=Color3.fromRGB(30,22,10),ON=Color3.fromRGB(180,140,40),OFF=Color3.fromRGB(80,60,20)},
    Green  = {A=Color3.fromRGB(80,220,120),A2=Color3.fromRGB(160,255,180),BG=Color3.fromRGB(10,24,16),BG2=Color3.fromRGB(16,36,22),BG3=Color3.fromRGB(14,28,20),ON=Color3.fromRGB(40,160,70),OFF=Color3.fromRGB(30,70,45)},
    Abyss  = {A=Color3.fromRGB(0,220,255),A2=Color3.fromRGB(150,240,255),BG=Color3.fromRGB(6,12,20),BG2=Color3.fromRGB(10,20,32),BG3=Color3.fromRGB(14,26,40),ON=Color3.fromRGB(0,180,200),OFF=Color3.fromRGB(40,50,66)},
}
local cur = "Abyss"
local A, A2, BG, BG2, BG3, ON, OFF
do
    local T = TH[cur]
    A,A2,BG,BG2,BG3,ON,OFF = T.A,T.A2,T.BG,T.BG2,T.BG3,T.ON,T.OFF
end

local FC = {}
(function() for k,_ in pairs(fl) do FC[k] = A end end)()
local SZ = { trail=1.5, parts=1.6, hat=1.4, foot=2.2, jump=4, bounce=0.5,
             orbAura=2.5, lightning=0.7, snow=1.2, glow=0.3, tracer=0.15,
             esp=0.5, ghost=0.3, atmo=0.35, rainbow=0.4, skyDust=1.5, bloom=1.4 }

local SID = "rbxassetid://159454299"
local HEART_MESH = "rbxassetid://6200353"

local jStyle, killOn, skyHeart, boHeart, timePreset = "Classic", true, false, false, "Day"
local killHooked = {}

local hatC, fsC, rbC, espC, jumpC, orbC, boC, ghC, skyC
local boFolder, skyFolder, cc, atmo, skyObj, blm

local target_p, target_char, target_part, target_hum
local nearest_player
local get_ws_fn
local install_hooks
local connect_trace_fn
local enableTracer_fn, disableTracer_fn
local applySpeed, applyJump, applyHip, applyGrav
local startInfJump, stopInfJump, startNoclip, stopNoclip
local startFly, stopFly, startSpin, stopSpin, moveReset
local troll_tick
local applyTime, toggleFog
local ka_tick
local hookKills
local bt_destroy, bt_build, bt_update, bt_conn

-- ============================================================
-- ROLE CACHE (survives ghost/death so ESP keeps color)
-- ============================================================
local roleCacheMod = nil
local roleCache = {}
local roleCacheStamp = 0

local function refreshRoleCache()
    local now = os.clock()
    if now - roleCacheStamp < 0.5 and next(roleCache) then return end
    roleCacheStamp = now
    if not roleCacheMod then
        local ok, m = pcall(function()
            return require(Replicated:WaitForChild("Modules"):WaitForChild("CurrentRoundClient"))
        end)
        if ok and type(m) == "table" then roleCacheMod = m end
    end
    local d = roleCacheMod and roleCacheMod.PlayerData
    if type(d) ~= "table" then return end
    local new = {}
    for name, info in pairs(d) do
        if type(info) == "table" and info.Role then
            new[name] = info.Role
        end
    end
    roleCache = new
end

-- ============================================================
-- HELPERS
-- ============================================================
local function alive(p)
    local c = p.Character; if not c then return false end
    local h = c:FindFirstChildOfClass("Humanoid")
    return h and h.Health > 0
end
local function roleOf(p)
    local c = p.Character; if not c then return "Innocent" end
    local b = p:FindFirstChild("Backpack")
    if c:FindFirstChild("Knife") or (b and b:FindFirstChild("Knife")) then return "Murderer" end
    if c:FindFirstChild("Gun")   or (b and b:FindFirstChild("Gun"))   then return "Sheriff" end
    return "Innocent"
end
local function getRole(pl)
    local bp = pl:FindFirstChildOfClass("Backpack")
    local ch = pl.Character
    local function chk(t)
        if not t then return nil end
        if t:FindFirstChild("Knife") then return "m" end
        if t:FindFirstChild("Gun")   then return "s" end
        return nil
    end
    return chk(bp) or chk(ch)
end

-- cachedRole: prefers PlayerData cache so ghost murderer stays red
local function cachedRole(pl)
    refreshRoleCache()
    local cached = roleCache[pl.Name]
    if cached == "Murderer" then return "m" end
    if cached == "Sheriff" or cached == "Hero" then return "s" end
    return getRole(pl)
end

local function hum()  local c=LP.Character; return c and c:FindFirstChildOfClass("Humanoid") end
local function HRP()  local c=LP.Character; return c and c:FindFirstChild("HumanoidRootPart") end
local function UP()   local c=LP.Character; return c and (c:FindFirstChild("UpperTorso") or c:FindFirstChild("Torso")) end
local function HEAD() local c=LP.Character; return c and c:FindFirstChild("Head") end
local function LT(c)  return Color3.new(math.min(1,c.R*1.4),math.min(1,c.G*1.4),math.min(1,c.B*1.4)) end

-- ============================================================
-- TARGETING
-- ============================================================
local rp = RaycastParams.new()
rp.FilterType = Enum.RaycastFilterType.Exclude
local SNAP = 48
local snap_t, snap_p = table.create(SNAP,0), table.create(SNAP,Vector3.zero)
local snap_n, snap_i = 0, 0
local track = { part=nil, pos=nil, time=0, vel=Vector3.zero, ready=false }
local EC = { rtt=0, jitter=0, seen=false }

local function refresh_target()
    refreshRoleCache()
    local best, bd = nil, math.huge
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and alive(p) and cachedRole(p) == "m" then
            local h = p.Character and p.Character:FindFirstChild("Head")
            if h then
                local d = (h.Position - Cam.CFrame.Position).Magnitude
                if d < bd then bd = d; best = p end
            end
        end
    end
    if best ~= target_p then
        target_p = best
        target_char, target_part, target_hum = nil, nil, nil
    end
    if not best then return end
    local c = best.Character
    if c ~= target_char then
        target_char = c; target_part, target_hum = nil, nil
    end
    if not c then return end
    if not target_part or not target_part.Parent then
        target_part = c:FindFirstChild("HumanoidRootPart")
            or c:FindFirstChild("UpperTorso") or c:FindFirstChild("Torso")
    end
    if not target_hum or not target_hum.Parent then
        target_hum = c:FindFirstChildOfClass("Humanoid")
    end
end

nearest_player = function()
    local best, bd = nil, math.huge
    local myp = HRP(); if not myp then return nil end
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                local d = (hrp.Position - myp.Position).Magnitude
                if d < bd then bd = d; best = p end
            end
        end
    end
    return best
end

local function spush(now,pos)
    snap_i = snap_i%SNAP+1
    snap_t[snap_i]=now; snap_p[snap_i]=pos
    if snap_n<SNAP then snap_n=snap_n+1 end
end
local function sget(k)
    local i=(snap_i-k-1)%SNAP+1
    return snap_t[i], snap_p[i]
end
local function fit_vel()
    if snap_n<3 then return nil end
    local newest=sget(0)
    local used,sd=0,0
    for k=0,snap_n-1 do
        local t=sget(k)
        if newest-t>0.5 then break end
        used=used+1; sd=sd+(t-newest)
    end
    if used<3 then return nil end
    local m=sd/used
    local num,den=Vector3.zero,0
    for k=0,used-1 do
        local t,p=sget(k)
        local d=(t-newest)-m
        num=num+p*d; den=den+d*d
    end
    if den<1e-8 then return nil end
    return num/den
end
local function track_clear()
    track.part=nil; track.pos=nil
    track.vel=Vector3.zero; track.ready=false
    snap_n,snap_i=0,0
end
local function track_step(now)
    local p=target_part
    if not p or not p.Parent then
        if track.part then track_clear() end
        return
    end
    local pos=p.Position
    if p~=track.part or not track.pos then
        track_clear(); track.part=p; track.pos=pos; track.time=now; spush(now,pos); return
    end
    local dt=now-track.time
    if dt>0.5 or (pos-track.pos).Magnitude>100 then
        track_clear(); track.part=p; track.pos=pos; track.time=now; spush(now,pos); return
    end
    if dt<=0 or (pos-track.pos).Magnitude==0 then return end
    spush(now,pos); track.pos=pos; track.time=now
    local v=fit_vel()
    if v then track.vel=v; track.ready=true end
end
local function sample_ping()
    local ok,ms = pcall(function()
        return Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
    end)
    local rtt = (ok and type(ms)=="number" and ms>0) and (ms/1000) or 0
    rtt = math.clamp(rtt,0,1)
    if EC.seen then
        EC.jitter = EC.jitter*0.9 + math.abs(rtt-EC.rtt)*0.1
        EC.rtt    = EC.rtt*0.85 + rtt*0.15
    else
        EC.rtt=rtt; EC.jitter=0; EC.seen=true
    end
end
local function lead_time()
    if not EC.seen then return 0 end
    return math.clamp(EC.rtt + EC.jitter*0.5, 0, 0.5)
end
local function predict(part, span)
    span = math.min(span or 0, 0.3)
    if not track.ready or (os.clock() - (track.time or 0)) > 0.15 then
        return part.Position
    end
    local g = workspace.Gravity
    local vel = track.vel
    local base = part.Position
    return Vector3.new(
        base.X + vel.X*span,
        base.Y + (track.ready and (vel.Y*span - 0.5*g*span*span) or 0),
        base.Z + vel.Z*span
    )
end
local HITS = {
    "HumanoidRootPart","UpperTorso","Torso","LowerTorso","Head",
    "LeftUpperArm","RightUpperArm","LeftLowerArm","RightLowerArm",
    "LeftUpperLeg","RightUpperLeg","LeftLowerLeg","RightLowerLeg",
    "LeftHand","RightHand","LeftFoot","RightFoot",
    "Left Arm","Right Arm","Left Leg","Right Leg",
}
local function get_aim_point(origin, use_predict)
    local c = target_char; if not c then return nil end
    local span = use_predict and lead_time() or 0
    local best
    local hrp0 = c:FindFirstChild("HumanoidRootPart")
    local minY = hrp0 and (hrp0.Position.Y - 3) or nil
    for _, name in ipairs(HITS) do
        local part = c:FindFirstChild(name)
        if part and part:IsA("BasePart") then
            local p = span>0 and predict(part,span) or part.Position
            if (p - part.Position).Magnitude > 15 then p = part.Position end
            if minY and p.Y < minY then p = Vector3.new(p.X, minY, p.Z) end
            best = best or p
            if origin then
                local dir = p - origin
                rp.FilterDescendantsInstances = { LP.Character }
                local hit = workspace:Raycast(origin,dir,rp)
                if not hit or hit.Instance:IsDescendantOf(c) then return p end
            else
                return p
            end
        end
    end
    return SA.walls and best or nil
end

-- ============================================================
-- WEAPON HOOK
-- ============================================================
local WSH = {}
get_ws_fn = function()
    if WSH.m then return WSH.m end
    local ok, m = pcall(function()
        return require(Replicated:WaitForChild("ClientServices"):WaitForChild("WeaponService"))
    end)
    if ok and type(m)=="table" then WSH.m=m end
    return WSH.m
end
local function origin_cf()
    local c = LP.Character
    local hrp = c and c:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil end
    local att = hrp:FindFirstChild("GunRaycastAttachment")
    if att then return att.WorldCFrame end
    return hrp.CFrame
end
local function resolve()
    if not SA.enabled then return nil end
    if not target_part or not target_part.Parent then
        pcall(refresh_target)
    end
    if not target_part or not target_part.Parent then
        local c = target_char
        local f = c and (c:FindFirstChild("Head") or c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("UpperTorso") or c:FindFirstChild("Torso"))
        if f and f.Parent then target_part = f else return nil end
    end
    local cf = origin_cf()
    local point = get_aim_point(cf and cf.Position, SA.predict)
    if not point then return nil end
    return CFrame.new(point)
end
install_hooks = function()
    local m = get_ws_fn(); if not m then return end
    pcall(function() setreadonly(m,false) end)
    if type(m.GetMouseTargetCFrame)=="function" and not WSH.om then
        WSH.om = m.GetMouseTargetCFrame
        m.GetMouseTargetCFrame = function(self,...)
            local cf = resolve()
            if cf then return cf end
            return WSH.om(self,...)
        end
    end
    if type(m.GetTargetPosition)=="function" and not WSH.os then
        WSH.os = m.GetTargetPosition
        m.GetTargetPosition = function(self,x,y,...)
            local cf = resolve()
            if cf then return cf end
            return WSH.os(self,x,y,...)
        end
    end
    if hookmetamethod and getnamecallmethod and newcclosure and not ADDON.ncHooked then
        pcall(function()
            local nc = newcclosure or function(f) return f end
            local old
            old = hookmetamethod(game, "__namecall", nc(function(self, ...)
                if not (SA.enabled and SA.walls) then return old(self, ...) end
                if ADDON.wbBusy then return old(self, ...) end
                local method
                local okm, mm = pcall(getnamecallmethod)
                if okm then method = mm end
                local isRay = method == "Raycast"
                local isOld = method == "FindPartOnRay" or method == "FindPartOnRayWithWhitelist" or method == "FindPartOnRayWithIgnoreList"
                local a1, a2, a3, a4, a5 = ...
                if (isRay or isOld) and self == workspace then
                    local origin, direction
                    if isRay then
                        origin, direction = ...
                    else
                        local ray = ...
                        if typeof(ray) == "Ray" then origin, direction = ray.Origin, ray.Direction end
                    end
                    if origin and direction and typeof(origin) == "Vector3" and typeof(direction) == "Vector3" and direction.Magnitude > 3 then
                        ADDON.wbBusy = true
                        local done = false
                        local o1, o2, o3, o4 = nil, nil, nil, nil
                        pcall(function()
                            local c0 = LP.Character
                            local h0 = c0 and c0:FindFirstChild("HumanoidRootPart")
                            local tc = target_char
                            if not (h0 and tc and target_part and target_part.Parent) then return end
                            if (origin - h0.Position).Magnitude > 120 then return end
                            if direction.Magnitude < 20 and direction.Y < 0 then return end
                            local p = get_aim_point(origin, SA.predict)
                            if not p then p = target_part.Position end
                            local dir = (p - origin)
                            if dir.Magnitude <= 1 or dir.Magnitude > 250 then return end
                            local ol = direction.Magnitude
                            if ol > 0.001 and direction.Unit:Dot(dir.Unit) < 0.9 then return end
                            local orig = table.pack(pcall(old, self, a1, a2, a3, a4, a5))
                            if orig[1] then
                                if isRay then
                                    local r0 = orig[2]
                                    if r0 and r0.Instance and (r0.Instance == tc or r0.Instance:IsDescendantOf(tc)) then
                                        done, o1, o2, o3, o4 = true, table.unpack(orig, 2, orig.n)
                                        return
                                    end
                                else
                                    local part0 = orig[2]
                                    if part0 and (part0 == tc or (typeof(part0) == "Instance" and part0:IsDescendantOf(tc))) then
                                        done, o1, o2, o3, o4 = true, table.unpack(orig, 2, orig.n)
                                        return
                                    end
                                end
                            end
                            local wp = RaycastParams.new()
                            wp.FilterType = Enum.RaycastFilterType.Include
                            wp.FilterDescendantsInstances = { tc }
                            wp.IgnoreWater = true
                            local r = nil
                            pcall(function() r = workspace:Raycast(origin, dir.Unit * 5000, wp) end)
                            if r and r.Instance then
                                if isRay then done, o1 = true, r
                                else done, o1, o2, o3, o4 = true, r.Instance, r.Position, r.Normal, r.Material end
                                return
                            end
                            if orig[1] then done, o1, o2, o3, o4 = true, table.unpack(orig, 2, orig.n) end
                        end)
                        ADDON.wbBusy = false
                        if done then return o1, o2, o3, o4 end
                    end
                end
                return old(self, ...)
            end))
            if type(old) ~= "function" then error("wb0") end
            ADDON.ncHooked = true
        end)
    end
end

getgenv().KNIFE_AIM_RESOLVE = function()
    if not KS.enabled then return nil end
    local c = LP.Character
    if not c or not c:FindFirstChild("Knife") then return nil end
    local part = target_part
    if not part or not part.Parent then return nil end
    local span = lead_time()
    local p = span>0 and predict(part,span) or part.Position
    return CFrame.new(p)
end

-- ============================================================
-- KILL AURA
-- ============================================================
ka_tick = function()
    if not KA.on then return end
    local c = LP.Character; if not c then return end
    local knife = c:FindFirstChild("Knife")
    if not knife then
        local bp = LP:FindFirstChildOfClass("Backpack")
        knife = bp and bp:FindFirstChild("Knife")
        if knife then
            local h = c:FindFirstChildOfClass("Humanoid")
            if h then pcall(function() h:EquipTool(knife) end) end
        end
        return
    end
    local now = os.clock()
    if now - KA.last < 0.05 then return end
    local myp = c:FindFirstChild("HumanoidRootPart"); if not myp then return end
    local events = knife:FindFirstChild("Events")
    local stabbed = events and events:FindFirstChild("KnifeStabbed")
    local touched = events and events:FindFirstChild("HandleTouched")
    if not stabbed or not touched then return end
    local fired = false
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP and alive(plr) then
            local tchar = plr.Character
            local tpart = tchar and (tchar:FindFirstChild("HumanoidRootPart") or tchar:FindFirstChild("Head"))
            if tpart and (tpart.Position - myp.Position).Magnitude <= KA.dist then
                if not fired then pcall(function() stabbed:FireServer() end); fired=true end
                pcall(function() touched:FireServer(tpart) end)
            end
        end
    end
    if fired then KA.last = now end
end

-- ============================================================
-- AUTO GET GUNDROP (exact shitaro version)
-- ============================================================
local function grab_has_role()
    if not grab_round_mod then
        local ok, m = pcall(function()
            return require(Replicated:WaitForChild("Modules"):WaitForChild("CurrentRoundClient"))
        end)
        if ok and type(m) == "table" then grab_round_mod = m end
    end
    local d = grab_round_mod and grab_round_mod.PlayerData
    if type(d) ~= "table" then return false end
    local me = d[LP.Name]
    return me ~= nil and me.Role ~= nil and not me.Dead
end

local function grab_has_knife()
    local char = LP.Character
    if char and char:FindFirstChild("Knife") then return true end
    local backpack = LP:FindFirstChildOfClass("Backpack")
    if backpack and backpack:FindFirstChild("Knife") then return true end
    return false
end

local function grab_gun(obj)
    if not autograb_on or grab_has_knife() then return end
    if not grab_has_role() then return end
    local char = LP.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    pcall(function() obj.CFrame = root.CFrame end)
    local prompt = obj:FindFirstChildOfClass("ProximityPrompt")
    if prompt then
        pcall(function() fireproximityprompt(prompt) end)
    end
end

local function scan_guns()
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj.Name == "GunDrop" and obj:IsA("BasePart") then
            grab_gun(obj)
        end
    end
end

local function grab_desc_added(obj)
    if obj.Name == "GunDrop" and obj:IsA("BasePart") then
        task.wait(0.1)
        grab_gun(obj)
    end
end

local function grab_desc_stop()
    if grab_desc_conn then
        pcall(function() grab_desc_conn:Disconnect() end)
        grab_desc_conn = nil
    end
end

-- ============================================================
-- BULLET TRACER
-- ============================================================
local function mkpt(p, life)
    local part = Instance.new("Part")
    part.Transparency=1; part.Anchored=true
    part.CanCollide=false; part.CanQuery=false
    part.Size=Vector3.new(1,1,1); part.CFrame=CFrame.new(p)
    Instance.new("Attachment", part)
    Debris:AddItem(part, life)
    part.Parent=workspace
    return part
end
local function draw_tracer(a,b)
    local dur = TRACE.dur
    local sp = mkpt(a,dur+0.5)
    local ep = mkpt(b,dur+0.5)
    local beam = Instance.new("Beam")
    beam.FaceCamera=true
    beam.Width0=0.1+SZ.tracer; beam.Width1=0.1+SZ.tracer
    beam.LightEmission=3; beam.LightInfluence=0; beam.Brightness=2.5
    beam.Texture="rbxassetid://12781800668"
    beam.TextureSpeed=1.5; beam.TextureLength=2
    beam.Color=ColorSequence.new(FC.tracer)
    beam.Transparency=NumberSequence.new(0.1)
    beam.Attachment0=sp.Attachment
    beam.Attachment1=ep.Attachment
    beam.Parent=sp
    task.delay(dur, function()
        if beam.Parent then
            Tween:Create(beam,TweenInfo.new(0.2),{Width0=0,Width1=0}):Play()
        end
    end)
end
local function to_pos(v)
    if typeof(v)=="Vector3" then return v end
    if typeof(v)=="CFrame" then return v.Position end
    if typeof(v)=="Instance" then
        if v:IsA("Attachment") then return v.WorldPosition end
        if v:IsA("BasePart") then return v.Position end
    end
    return nil
end
connect_trace_fn = function()
    if TRACE.conn then return end
    local m = get_ws_fn()
    local ev = m and m.GunFired
    if typeof(ev)~="Instance" then return end
    TRACE.conn = ev.OnClientEvent:Connect(function(gun,a,b)
        if not TRACE.on then return end
        local c = LP.Character; if not c then return end
        if typeof(gun)~="Instance" or not gun:IsDescendantOf(c) then return end
        local pa,pb = to_pos(a), to_pos(b)
        if pa and pb then task.spawn(draw_tracer, pa, pb) end
    end)
end

-- ============================================================
-- BACKTRACK
-- ============================================================
bt_destroy = function()
    if BT.model then
        if _G.UN_BT_CLONES then _G.UN_BT_CLONES[BT.model] = nil end
        pcall(function() BT.model:Destroy() end)
        BT.model = nil
    end
    BT.cache = {}; BT.hist = {}; BT.first = 1; BT.n = 0
end

bt_build = function()
    bt_destroy()
    local char = LP.Character
    if not char then return end
    local rhrp = char:FindFirstChild("HumanoidRootPart")
    if not rhrp then return end
    char.Archivable = true
    local ok, m = pcall(function() return char:Clone() end)
    char.Archivable = false
    if not ok or not m then return end
    _G.UN_BT_CLONES = _G.UN_BT_CLONES or {}
    local rparts = {}
    for _, o in char:GetDescendants() do
        if o:IsA("BasePart") then rparts[#rparts + 1] = o end
    end
    local ci = 0
    for _, o in m:GetDescendants() do
        if o:IsA("Script") or o:IsA("LocalScript") or o:IsA("ModuleScript")
            or o:IsA("Decal") or o:IsA("Texture") or o:IsA("SurfaceAppearance")
            or o:IsA("ParticleEmitter") or o:IsA("Beam") or o:IsA("Trail")
            or o:IsA("PointLight") or o:IsA("SpotLight") or o:IsA("SurfaceLight")
            or o:IsA("Sound") or o:IsA("Fire") or o:IsA("Smoke") or o:IsA("Sparkles") then
            pcall(function() o:Destroy() end)
        elseif o:IsA("BasePart") then
            o.Anchored = true; o.CanCollide = false; o.CanQuery = false
            o.CanTouch = false; o.Massless = true; o.CastShadow = false; o.Locked = true
            if o.Name == "HumanoidRootPart" then
                o.Transparency = 1
            else
                o.Material = Enum.Material.ForceField
                o.Color = Color3.fromRGB(170, 100, 255)
                o.Transparency = 0
            end
            ci = ci + 1
            BT.cache[#BT.cache + 1] = {o, rparts[ci]}
        end
    end
    local hum2 = m:FindFirstChildOfClass("Humanoid")
    if hum2 then pcall(function() hum2:Destroy() end) end
    m.Name = "\0"
    m.Parent = workspace
    BT.model = m
    _G.UN_BT_CLONES[m] = true
end

bt_update = function()
    if not BT.on then return end
    local char = LP.Character
    local rhrp = char and char:FindFirstChild("HumanoidRootPart")
    if not rhrp then return end
    if not BT.model or not BT.model.Parent then
        bt_build()
        if not BT.model then return end
    end
    local now = os.clock()
    local base_cf = rhrp.CFrame
    if BT.n < BT.cap then BT.n = BT.n + 1
    else BT.first = BT.first % BT.cap + 1 end
    local idx = (BT.first + BT.n - 2) % BT.cap + 1
    local slot = BT.hist[idx]
    if not slot then slot = {}; BT.hist[idx] = slot end
    slot[1] = now
    slot[2] = base_cf

    if now - BT.lastping >= 0.2 then
        BT.lastping = now
        local ok, val = pcall(function()
            return Stats.Network.ServerStatsItem["Data Ping"]:GetValue() / 1000
        end)
        BT.ping = math.clamp((ok and val) or 0.15, 0.05, 0.6)
    end
    local target = now - BT.ping
    local cf = base_cf
    for k = BT.n, 1, -1 do
        local s = BT.hist[(BT.first + k - 2) % BT.cap + 1]
        if s and s[1] and s[1] <= target then cf = s[2]; break end
    end
    while BT.n > 0 and BT.hist[BT.first] and BT.hist[BT.first][1] < now - 1 do
        BT.first = BT.first % BT.cap + 1
        BT.n = BT.n - 1
    end
    local baseInv = rhrp.CFrame:Inverse()
    for i = 1, #BT.cache do
        local cp, rpv = BT.cache[i][1], BT.cache[i][2]
        if cp and cp.Parent and rpv and rpv.Parent then
            cp.CFrame = cf * (baseInv * rpv.CFrame)
        end
    end
end

bt_conn = RS.Heartbeat:Connect(function()
    if BT.on then pcall(bt_update) end
end)

-- ============================================================
-- MOVE
-- ============================================================
applySpeed = function() local h=hum(); if h then h.WalkSpeed = MOVE.speedOn and MOVE.speed or 16 end end
applyJump  = function() local h=hum(); if h then h.JumpPower = MOVE.jumpOn and MOVE.jump or 50 end end
applyHip   = function() local h=hum(); if h then h.HipHeight = MOVE.hipOn and MOVE.hip or 2 end end
applyGrav  = function() workspace.Gravity = MOVE.gravOn and MOVE.grav or 196.2 end

startInfJump = function()
    if moveConns.infJump then return end
    moveConns.infJump = UIS.JumpRequest:Connect(function()
        if not MOVE.infJump then return end
        local h = hum(); if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
    end)
end
stopInfJump = function()
    if moveConns.infJump then moveConns.infJump:Disconnect(); moveConns.infJump=nil end
end

startNoclip = function()
    if moveConns.noclip then return end
    table.clear(moveConns.nc)
    moveConns.noclip = RS.Stepped:Connect(function()
        if not MOVE.noclip or not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        for _, p in ipairs(c:GetDescendants()) do
            if p:IsA("BasePart") then
                if moveConns.nc[p] == nil then moveConns.nc[p] = p.CanCollide end
                if p.CanCollide then p.CanCollide=false end
            end
        end
    end)
end
stopNoclip = function()
    if moveConns.noclip then moveConns.noclip:Disconnect(); moveConns.noclip=nil end
    for p, was in pairs(moveConns.nc) do
        if p and p.Parent then pcall(function() p.CanCollide = was end) end
    end
    table.clear(moveConns.nc)
    local c = LP.Character
    if c then
        local hrp = c:FindFirstChild("HumanoidRootPart")
        if hrp then pcall(function() hrp.CanCollide = false end) end
    end
end

startFly = function()
    if flyConn then pcall(function() flyConn:Disconnect() end) flyConn = nil end
    if MOVE.fg then pcall(function() MOVE.fg:Destroy() end) MOVE.fg = nil end
    if MOVE.fv then pcall(function() MOVE.fv:Destroy() end) MOVE.fv = nil end
    local c0 = LP.Character
    local hrp0 = c0 and c0:FindFirstChild("HumanoidRootPart")
    if not hrp0 then return end
    MOVE.fg = Instance.new("BodyGyro")
    MOVE.fg.P = 9e4 MOVE.fg.D = 500 MOVE.fg.MaxTorque = Vector3.new(9e9,9e9,9e9)
    MOVE.fg.CFrame = workspace.CurrentCamera.CFrame MOVE.fg.Parent = hrp0
    MOVE.fv = Instance.new("BodyVelocity")
    MOVE.fv.MaxForce = Vector3.new(9e9,9e9,9e9) MOVE.fv.Velocity = Vector3.zero MOVE.fv.Parent = hrp0
    if MOVE.upBtn then MOVE.upBtn.Visible = true end
    if MOVE.dnBtn then MOVE.dnBtn.Visible = true end
    flyConn = RS.RenderStepped:Connect(function(dt)
        if not MOVE.fly or not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        local h = c:FindFirstChildOfClass("Humanoid")
        if not hrp or not h then return end
        pcall(function() h.PlatformStand = true end)
        local cam = workspace.CurrentCamera
        if MOVE.fg and MOVE.fg.Parent then MOVE.fg.CFrame = cam.CFrame end
        local dir = Vector3.zero
        local fwd = (cam.CFrame.LookVector * Vector3.new(1,0,1))
        if fwd.Magnitude > 0.01 then fwd = fwd.Unit else fwd = Vector3.zero end
        local rgt = (cam.CFrame.RightVector * Vector3.new(1,0,1))
        if rgt.Magnitude > 0.01 then rgt = rgt.Unit else rgt = Vector3.zero end
        if UIS:IsKeyDown(Enum.KeyCode.W) then dir = dir + cam.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.S) then dir = dir - cam.CFrame.LookVector end
        if UIS:IsKeyDown(Enum.KeyCode.A) then dir = dir - rgt end
        if UIS:IsKeyDown(Enum.KeyCode.D) then dir = dir + rgt end
        if UIS:IsKeyDown(Enum.KeyCode.Space) or UIS:IsKeyDown(Enum.KeyCode.E) then dir = dir + Vector3.new(0,1,0) end
        if UIS:IsKeyDown(Enum.KeyCode.LeftControl) or UIS:IsKeyDown(Enum.KeyCode.Q) then dir = dir - Vector3.new(0,1,0) end
        if MOVE.flyUp then dir = dir + Vector3.new(0,1,0) end
        if MOVE.flyDown then dir = dir - Vector3.new(0,1,0) end
        local mv = h.MoveDirection
        if dir.Magnitude < 0.1 and mv.Magnitude > 0.1 then
            dir = fwd * -mv.Z + rgt * mv.X
            if UIS:IsKeyDown(Enum.KeyCode.Space) then dir = dir + Vector3.new(0,1,0) end
        end
        local step = math.min(dt or 0.016, 0.05)
        local vel = Vector3.zero
        if dir.Magnitude > 0.1 then
            vel = dir.Unit * MOVE.flySpeed
            hrp.CFrame = hrp.CFrame + dir.Unit * MOVE.flySpeed * step
        end
        if MOVE.fv and MOVE.fv.Parent then MOVE.fv.Velocity = vel end
        pcall(function()
            hrp.RotVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end)
    end)
end
stopFly = function()
    if flyConn then flyConn:Disconnect(); flyConn=nil end
    if MOVE.fg then pcall(function() MOVE.fg:Destroy() end) MOVE.fg = nil end
    if MOVE.fv then pcall(function() MOVE.fv:Destroy() end) MOVE.fv = nil end
    if MOVE.upBtn then MOVE.upBtn.Visible = false end
    if MOVE.dnBtn then MOVE.dnBtn.Visible = false end
    MOVE.flyUp, MOVE.flyDown = false, false
    local c = LP.Character
    if c then
        local h = c:FindFirstChildOfClass("Humanoid")
        if h then pcall(function() h.PlatformStand=false end) pcall(function() h:ChangeState(Enum.HumanoidStateType.GettingUp) end) end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        if hrp then pcall(function()
            hrp.Velocity = Vector3.zero
            hrp.RotVelocity = Vector3.zero
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end) end
    end
end

startSpin = function()
    if spinConn then return end
    spinConn = RS.RenderStepped:Connect(function(dt)
        if not MOVE.spin or not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart"); if not hrp then return end
        hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(MOVE.spinSpeed*dt), 0)
    end)
end
stopSpin = function()
    if spinConn then spinConn:Disconnect(); spinConn=nil end
end

moveReset = function()
    MOVE.speedOn=false; MOVE.jumpOn=false; MOVE.hipOn=false
    MOVE.infJump=false; MOVE.noclip=false; MOVE.fly=false
    MOVE.gravOn=false; MOVE.spin=false
    MOVE.antiFall=false
    stopInfJump(); stopNoclip(); stopFly(); stopSpin()
    applySpeed(); applyJump(); applyHip(); applyGrav()
end
task.spawn(function()
    local lastSafe = nil
    while ADDON.loaded do
        pcall(function()
            local c = LP.Character
            local hrp = c and c:FindFirstChild("HumanoidRootPart")
            if hrp then
                if hrp.Position.Y > workspace.FallenPartsDestroyHeight + 30 then
                    lastSafe = hrp.CFrame
                elseif MOVE.antiFall and lastSafe then
                    hrp.CFrame = lastSafe
                    hrp.Velocity = Vector3.zero
                end
            end
        end)
        task.wait(0.2)
    end
end)

task.spawn(function()
    while ADDON.loaded do
        pcall(function()
            if MOVE.fly and not flyConn then startFly() end
            if MOVE.spin and not spinConn then startSpin() end
            if MOVE.noclip and not moveConns.noclip then startNoclip() end
            if MOVE.infJump and not moveConns.infJump then startInfJump() end
        end)
        pcall(function() Cam = workspace.CurrentCamera end)
        if not TROLL.flinging then
            local cc = workspace.CurrentCamera
            local mc = LP.Character
            local mh = mc and mc:FindFirstChildOfClass("Humanoid")
            if cc and mh and mh.Health > 0 then
                local subj = cc.CameraSubject
                local bad = subj == nil
                if not bad and typeof(subj) == "Instance" then
                    if not subj.Parent then bad = true
                    elseif subj:IsA("Humanoid") then
                        if subj.Health <= 0 then bad = true
                        else
                            local m = subj:FindFirstAncestorOfClass("Model")
                            if m and m ~= mc and subj ~= mh then bad = true end
                        end
                    elseif subj:IsA("BasePart") then
                        local m = subj:FindFirstAncestorOfClass("Model")
                        if m and m ~= mc then bad = true end
                    else
                        bad = true
                    end
                end
                if bad then pcall(function() cc.CameraSubject = mh end) end
            end
        end
        task.wait(1)
    end
end)

-- ============================================================
-- TROLL
-- ============================================================
local function flingCore(thrp, thum, time)
    local c = LP.Character
    local hrp = c and c:FindFirstChild("HumanoidRootPart"); if not hrp then return false end
    local hum = c and c:FindFirstChildOfClass("Humanoid"); if not hum then return false end
    if thum and thum.Sit then return false end
    local orig = hrp.CFrame
    local cam = workspace.CurrentCamera
    local oldSubj = cam.CameraSubject
    local oldFall = workspace.FallenPartsDestroyHeight
    pcall(function() workspace.FallenPartsDestroyHeight = 0/0 end)
    pcall(function() cam.CameraSubject = thrp end)
    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(9e9,9e9,9e9) bv.Velocity = Vector3.zero bv.Parent = hrp
    local se = hum:GetStateEnabled(Enum.HumanoidStateType.Seated)
    pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Seated, false) end)
    local t0 = os.clock()
    local dur = time or 1.6
    local ang = 0
    while os.clock() - t0 < dur and hrp.Parent and thrp.Parent do
        local tv = thrp.Velocity
        if tv.Magnitude > 50 and thum then tv = thum.MoveDirection * thum.WalkSpeed end
        local lead = thrp.Position + tv * 0.08
        ang = ang + 165
        local r = ang % 360
        local off
        if r < 90 then off = Vector3.new(0,2.5,0)
        elseif r < 180 then off = Vector3.new(0,-2.5,0)
        elseif r < 270 then off = Vector3.new(2.5,0,0)
        else off = Vector3.new(-2.5,0,0) end
        local cf = CFrame.new(lead + off) * CFrame.Angles(math.rad(ang),0,0)
        pcall(function()
            hrp.CFrame = cf
            c:SetPrimaryPartCFrame(cf)
            hrp.Velocity = Vector3.new(9e8,9e8*10,9e8)
            hrp.RotVelocity = Vector3.new(9e9,9e9,9e9)
            hrp.AssemblyLinearVelocity = Vector3.new(9e8,9e8*10,9e8)
            hrp.AssemblyAngularVelocity = Vector3.new(9e9,9e9,9e9)
        end)
        RS.Heartbeat:Wait()
    end
    if bv.Parent then bv:Destroy() end
    pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Seated, se) end)
    pcall(function() cam.CameraSubject = hum end)
    if hrp.Parent then
        pcall(function()
            hrp.CFrame = orig
            c:SetPrimaryPartCFrame(orig)
            hum:ChangeState(Enum.HumanoidStateType.GettingUp)
            hrp.Velocity = Vector3.zero
            hrp.RotVelocity = Vector3.zero
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end)
    end
    pcall(function() workspace.FallenPartsDestroyHeight = oldFall end)
    if oldSubj then pcall(function() cam.CameraSubject = oldSubj end) end
    return true
end
local function flingTarget(target)
    if TROLL.flinging then return end
    local tchar = target and target.Character; if not tchar then return end
    local thrp = tchar:FindFirstChild("HumanoidRootPart") or tchar:FindFirstChild("Head"); if not thrp then return end
    local thum = tchar:FindFirstChildOfClass("Humanoid")
    TROLL.flinging = true
    task.spawn(function()
        flingCore(thrp, thum, 2.2)
        TROLL.flinging = false
    end)
end
local function flingNearest()
    local t = nearest_player()
    if t then flingTarget(t) end
end
local function flingAll()
    if TROLL.flinging then return end
    TROLL.flinging = true
    task.spawn(function()
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local thrp = p.Character:FindFirstChild("HumanoidRootPart") or p.Character:FindFirstChild("Head")
                local thum = p.Character:FindFirstChildOfClass("Humanoid")
                if thrp and LP.Character and LP.Character:FindFirstChild("HumanoidRootPart") then
                    flingCore(thrp, thum, 1.5)
                    task.wait(0.15)
                end
            end
        end
        TROLL.flinging=false
    end)
end
local function stopTroll()
    TROLL.flinging=false
end
troll_tick = function()
    if TROLL.flinging then return end
end

-- ============================================================
-- VISUALS
-- ============================================================
local function mkTrail()
    local up = UP(); if not up then return end
    if up:FindFirstChild("sT0") then return end
    local col = FC.trail
    local a0 = Instance.new("Attachment", up); a0.Name="sT0"; a0.Position=Vector3.new(-1,0.5,-0.8)
    local a1 = Instance.new("Attachment", up); a1.Name="sT1"; a1.Position=Vector3.new(1,-0.5,-0.8)
    local t = Instance.new("Trail", up); t.Name="sTrail"
    t.Attachment0=a0; t.Attachment1=a1; t.Lifetime=0.5; t.MinLength=0.1
    t.WidthScale=NumberSequence.new({NumberSequenceKeypoint.new(0,SZ.trail),NumberSequenceKeypoint.new(1,0)})
    t.Color=ColorSequence.new(col, LT(col))
    t.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0),NumberSequenceKeypoint.new(0.6,0.3),NumberSequenceKeypoint.new(1,1)})
    t.LightEmission=1; t.LightInfluence=0; t.FaceCamera=true
end
local function clTrail()
    local c = LP.Character; if not c then return end
    for _,x in ipairs(c:GetDescendants()) do
        if x.Name=="sTrail" or x.Name=="sT0" or x.Name=="sT1" then x:Destroy() end
    end
end

local function mkParts()
    local hrp = HRP(); if not hrp then return end
    if hrp:FindFirstChild("sAuraAtt") then return end
    local col = FC.parts
    local att = Instance.new("Attachment", hrp); att.Name="sAuraAtt"
    local p = Instance.new("ParticleEmitter", att); p.Name="sAura"
    p.Texture="rbxasset://textures/particles/sparkles_main.dds"
    p.Rate=50; p.Lifetime=NumberRange.new(1.2,2); p.Speed=NumberRange.new(2,4)
    p.SpreadAngle=Vector2.new(180,180); p.Rotation=NumberRange.new(0,360); p.RotSpeed=NumberRange.new(-90,90)
    p.Size=NumberSequence.new({NumberSequenceKeypoint.new(0,0),NumberSequenceKeypoint.new(0.2,SZ.parts),NumberSequenceKeypoint.new(1,0)})
    p.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,1),NumberSequenceKeypoint.new(0.15,0),NumberSequenceKeypoint.new(1,1)})
    p.Color=ColorSequence.new(col, LT(col))
    p.LightEmission=1; p.LightInfluence=0; p.Drag=8; p.ZOffset=8
end
local function clParts()
    local hrp = HRP(); if not hrp then return end
    local x = hrp:FindFirstChild("sAuraAtt"); if x then x:Destroy() end
end

local function mkSnow()
    local hrp = HRP(); if not hrp then return end
    if hrp:FindFirstChild("sSnowAtt") then return end
    local col = FC.snow
    local att = Instance.new("Attachment", hrp); att.Name="sSnowAtt"
    local p = Instance.new("ParticleEmitter", att); p.Name="sSnow"
    p.Texture="rbxasset://textures/particles/sparkles_main.dds"
    p.Rate=30; p.Lifetime=NumberRange.new(2,3); p.Speed=NumberRange.new(3,6)
    p.SpreadAngle=Vector2.new(40,40)
    p.Size=NumberSequence.new({NumberSequenceKeypoint.new(0,SZ.snow),NumberSequenceKeypoint.new(1,0)})
    p.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0),NumberSequenceKeypoint.new(1,1)})
    p.Color=ColorSequence.new(col, LT(col))
    p.LightEmission=1; p.LightInfluence=0
    p.Acceleration=Vector3.new(0,-5,0); p.Drag=1; p.ZOffset=8
end
local function clSnow()
    local hrp = HRP(); if not hrp then return end
    local x = hrp:FindFirstChild("sSnowAtt"); if x then x:Destroy() end
end

local function mkSky()
    if skyC then return end
    if not skyFolder then
        skyFolder = Instance.new("Folder", workspace); skyFolder.Name="l00kSky"
    end
    local last = 0
    skyC = RS.Heartbeat:Connect(function()
        if not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart"); if not hrp then return end
        local n = tick(); if n-last<0.3 then return end
        last = n
        local col = FC.skyDust
        for _=1,2 do
            local off = Vector3.new((math.random()-0.5)*60, 30+math.random()*40, (math.random()-0.5)*60)
            local p = Instance.new("Part"); p.Material=Enum.Material.Neon
            if skyHeart then
                p.Size=Vector3.new(SZ.skyDust*2,SZ.skyDust*2,SZ.skyDust*2)
                p.Color=Color3.fromRGB(255,80,140)
                local m = Instance.new("SpecialMesh", p)
                m.MeshType=Enum.MeshType.FileMesh; m.MeshId=HEART_MESH
                m.Scale=Vector3.new(1.5,1.5,1.5)*SZ.skyDust
            else
                p.Shape=Enum.PartType.Ball
                p.Size=Vector3.new(SZ.skyDust,SZ.skyDust,SZ.skyDust)
                p.Color=col
            end
            p.CanCollide=false; p.CanQuery=false; p.CanTouch=false; p.CastShadow=false
            p.Anchored=true; p.Transparency=0.05
            p.CFrame=CFrame.new(hrp.Position + off); p.Parent=skyFolder
            local life, maxLife = 0, 6+math.random()*4
            local cn
            cn = RS.Heartbeat:Connect(function(dt)
                life=life+dt
                if life>=maxLife or not p.Parent then
                    cn:Disconnect(); if p.Parent then p:Destroy() end; return
                end
                p.CFrame = p.CFrame * CFrame.new((math.random()-0.5)*0.05, -15*dt, (math.random()-0.5)*0.05)
                p.Transparency = math.min(0.95, life/maxLife)
            end)
        end
    end)
end
local function clSky()
    if skyC then skyC:Disconnect(); skyC=nil end
    if skyFolder then skyFolder:Destroy(); skyFolder=nil end
end

local function mkLight()
    local hrp = HRP(); if not hrp then return end
    if hrp:FindFirstChild("sLightAtt") then return end
    local col = FC.lightning
    local a = Instance.new("Attachment", hrp); a.Name="sLightAtt"
    local p = Instance.new("ParticleEmitter", a); p.Name="sLight"
    p.Texture="rbxasset://textures/particles/sparkles_main.dds"
    p.Rate=15; p.Lifetime=NumberRange.new(0.2,0.5); p.Speed=NumberRange.new(10,18)
    p.SpreadAngle=Vector2.new(180,180)
    p.Size=NumberSequence.new({NumberSequenceKeypoint.new(0,SZ.lightning),NumberSequenceKeypoint.new(1,0)})
    p.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0),NumberSequenceKeypoint.new(1,1)})
    p.Color=ColorSequence.new(LT(col), col)
    p.LightEmission=1; p.LightInfluence=0; p.ZOffset=4
end
local function clLight()
    local hrp = HRP(); if not hrp then return end
    local x = hrp:FindFirstChild("sLightAtt"); if x then x:Destroy() end
end

local function mkBo()
    if boC then return end
    if not boFolder then
        boFolder = Instance.new("Folder", workspace); boFolder.Name="l00kB"
    end
    local last = 0
    boC = RS.Heartbeat:Connect(function()
        if not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        local h2 = c:FindFirstChildOfClass("Humanoid")
        if not hrp or not h2 then return end
        if not (h2.MoveDirection.Magnitude>0.1 and h2.FloorMaterial~=Enum.Material.Air) then return end
        local n = tick(); if n-last<0.15 then return end
        last = n
        local col = FC.bounce
        for _=1,2 do
            local p = Instance.new("Part"); p.Material=Enum.Material.Neon
            local baseSize = SZ.bounce
            if boHeart then
                p.Size=Vector3.new(baseSize*2,baseSize*2,baseSize*2)
                p.Color=Color3.fromRGB(255,80,140)
                local m = Instance.new("SpecialMesh", p)
                m.MeshType=Enum.MeshType.FileMesh; m.MeshId=HEART_MESH
                m.Scale=Vector3.new(1.5,1.5,1.5)*baseSize
            else
                p.Shape=Enum.PartType.Ball
                p.Size=Vector3.new(baseSize,baseSize,baseSize); p.Color=col
            end
            p.CanCollide=false; p.CanQuery=false; p.CanTouch=false; p.CastShadow=false
            p.Anchored=true
            p.CFrame = hrp.CFrame * CFrame.new((math.random()-0.5)*4, 1.5+math.random(), (math.random()-0.5)*4)
            p.Parent = boFolder
            local vel = Vector3.new((math.random()-0.5)*6, 3+math.random()*3, (math.random()-0.5)*6)
            local life = 0
            local rp2 = RaycastParams.new()
            rp2.FilterDescendantsInstances={c}
            rp2.FilterType = Enum.RaycastFilterType.Exclude
            local cn
            cn = RS.Heartbeat:Connect(function(dt)
                life=life+dt
                if not p.Parent then cn:Disconnect(); return end
                vel = vel + Vector3.new(0,-30*dt,0)
                local np = p.Position + vel*dt
                if vel.Y<0 then
                    local ray = workspace:Raycast(p.Position, Vector3.new(0,-6,0), rp2)
                    if ray and (p.Position.Y - ray.Position.Y) < 0.7 then
                        np = Vector3.new(np.X, ray.Position.Y + baseSize/2 + 0.1, np.Z)
                        vel = Vector3.new(vel.X, -vel.Y*0.6, vel.Z)
                    end
                end
                p.Position = np
                if life>=3 then cn:Disconnect(); p:Destroy() end
            end)
        end
    end)
end
local function clBo()
    if boC then boC:Disconnect(); boC=nil end
    if boFolder then boFolder:Destroy(); boFolder=nil end
end

local function mkHat()
    local c = LP.Character; if not c then return end
    local head = HEAD(); if not head then return end
    if c:FindFirstChild("sHat") then return end
    local col = FC.hat
    local h = Instance.new("Part")
    h.Name="sHat"; h.Size=Vector3.new(1.2,0.8,1.2)
    h.Massless=true; h.CanCollide=false; h.CanQuery=false; h.CanTouch=false
    h.Anchored=true; h.Material=Enum.Material.Neon; h.Color=col
    h.Transparency=0.05; h.CastShadow=false
    local m = Instance.new("SpecialMesh", h)
    m.MeshType=Enum.MeshType.FileMesh; m.MeshId="rbxassetid://1033714"
    m.Scale=Vector3.new(SZ.hat,SZ.hat,SZ.hat)
    h.Parent = c
    local r = Instance.new("Part")
    r.Name="sHatRing"; r.Shape=Enum.PartType.Cylinder
    r.Size=Vector3.new(0.1, 1.7*SZ.hat, 1.7*SZ.hat)
    r.Massless=true; r.CanCollide=false; r.CanQuery=false; r.CanTouch=false
    r.Anchored=true; r.Material=Enum.Material.Neon; r.Color=LT(col)
    r.Transparency=0.2; r.CastShadow=false; r.Parent=c
    if hatC then hatC:Disconnect() end
    hatC = RS.RenderStepped:Connect(function()
        if not h.Parent or not r.Parent then hatC:Disconnect(); return end
        if not head.Parent then return end
        h.CFrame = head.CFrame * CFrame.new(0,1.2,0)
        r.CFrame = head.CFrame * CFrame.new(0,0.75,0) * CFrame.Angles(0,0,math.rad(90))
    end)
end
local function clHat()
    if hatC then hatC:Disconnect(); hatC=nil end
    local c = LP.Character
    if c then
        for _,n in ipairs({"sHat","sHatRing"}) do
            local x = c:FindFirstChild(n); if x then x:Destroy() end
        end
    end
end

local function mkRB()
    local c = LP.Character; if not c then return end
    if c:FindFirstChild("sRB") then return end
    local h = Instance.new("Highlight")
    h.Name="sRB"; h.FillTransparency=0.35; h.OutlineTransparency=0
    h.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop; h.Parent=c
    local hue = 0
    if rbC then rbC:Disconnect() end
    rbC = RS.Heartbeat:Connect(function(dt)
        if not h.Parent then rbC:Disconnect(); return end
        hue = (hue + dt*SZ.rainbow) % 1
        local col = Color3.fromHSV(hue,0.85,1)
        h.FillColor=col; h.OutlineColor=col
    end)
end
local function clRB()
    if rbC then rbC:Disconnect(); rbC=nil end
    local c = LP.Character
    if c then local x=c:FindFirstChild("sRB"); if x then x:Destroy() end end
end

local function mkGlow()
    local c = LP.Character; if not c then return end
    if c:FindFirstChild("sGlow") then return end
    local h = Instance.new("Highlight")
    h.Name="sGlow"
    h.FillColor=FC.glow; h.OutlineColor=LT(FC.glow)
    h.FillTransparency=SZ.glow; h.OutlineTransparency=0.2
    h.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop
    h.Parent=c
end
local function clGlow()
    local c = LP.Character
    if c then local x=c:FindFirstChild("sGlow"); if x then x:Destroy() end end
end

local function mkFoot()
    if fsC then fsC:Disconnect() end
    local last = 0
    fsC = RS.Heartbeat:Connect(function()
        if not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        local h2 = c:FindFirstChildOfClass("Humanoid")
        if not hrp or not h2 then return end
        if h2.MoveDirection.Magnitude>0.1 and h2.FloorMaterial~=Enum.Material.Air then
            local n = tick(); if n-last<0.28 then return end
            last = n
            local col=FC.foot; local s=SZ.foot
            local p = Instance.new("Part")
            p.Shape=Enum.PartType.Cylinder
            p.Size=Vector3.new(0.15,s,s)
            p.Anchored=true; p.CanCollide=false; p.CanQuery=false; p.CanTouch=false
            p.CastShadow=false; p.Material=Enum.Material.Neon; p.Color=col; p.Transparency=0.15
            p.CFrame = CFrame.new(hrp.Position - Vector3.new(0,2.7,0)) * CFrame.Angles(0,0,math.rad(90))
            p.Parent = workspace
            local ts = 0
            local cn
            cn = RS.Heartbeat:Connect(function(dt)
                ts=ts+dt
                local k = ts/0.45
                if k>=1 or not p.Parent then cn:Disconnect(); if p.Parent then p:Destroy() end; return end
                local sz = s*(1-k*0.6)
                p.Size=Vector3.new(0.15,sz,sz)
                p.Transparency=0.15+k*0.85
            end)
        end
    end)
end
local function clFoot()
    if fsC then fsC:Disconnect(); fsC=nil end
end

local function mkOrb()
    if orbC then return end
    local hrp = HRP(); if not hrp then return end
    if hrp:FindFirstChild("sOrbs") then return end
    local f = Instance.new("Folder", hrp); f.Name="sOrbs"
    local col = FC.orbAura
    local orbs = {}
    for i=1,6 do
        local o = Instance.new("Part")
        o.Shape=Enum.PartType.Ball; o.Size=Vector3.new(0.35,0.35,0.35)
        o.Material=Enum.Material.Neon; o.Color=col
        o.CanCollide=false; o.CanQuery=false; o.CanTouch=false
        o.CastShadow=false; o.Anchored=true; o.Parent=f
        local gl = Instance.new("PointLight", o)
        gl.Color=col; gl.Brightness=1; gl.Range=5
        orbs[i]=o
    end
    local t = 0
    orbC = RS.Heartbeat:Connect(function(dt)
        t=t+dt
        if not hrp.Parent or not f.Parent then
            if orbC then orbC:Disconnect() end; orbC=nil
            if f.Parent then f:Destroy() end; return
        end
        for i,o in ipairs(orbs) do
            local a = t*2 + i*(math.pi/3)
            o.CFrame = hrp.CFrame * CFrame.new(math.cos(a)*SZ.orbAura, math.sin(t*3+i)*0.4, math.sin(a)*SZ.orbAura)
        end
    end)
end
local function clOrb()
    if orbC then orbC:Disconnect(); orbC=nil end
    local c = LP.Character
    if c then
        local hrp = c:FindFirstChild("HumanoidRootPart")
        if hrp then local f=hrp:FindFirstChild("sOrbs"); if f then f:Destroy() end end
    end
end

local function mkGhost()
    if ghC then return end
    local last = 0
    ghC = RS.Heartbeat:Connect(function()
        if not ADDON.loaded then return end
        local c = LP.Character; if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        local h2 = c:FindFirstChildOfClass("Humanoid")
        if not hrp or not h2 then return end
        if h2.MoveDirection.Magnitude<0.1 then return end
        local n = tick(); if n-last<0.12 then return end
        last = n
        local ghostH = Instance.new("Highlight")
        ghostH.Name="sGhost"; ghostH.Adornee=c
        ghostH.FillColor=FC.ghost; ghostH.OutlineColor=FC.ghost
        ghostH.FillTransparency=0.4; ghostH.OutlineTransparency=0
        ghostH.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop
        ghostH.Parent=workspace
        local t = 0
        local cn
        cn = RS.Heartbeat:Connect(function(dt)
            t=t+dt
            local k=t/0.4
            if k>=1 or not ghostH.Parent then cn:Disconnect(); if ghostH.Parent then ghostH:Destroy() end; return end
            ghostH.FillTransparency = 0.4 + k*0.55
            ghostH.OutlineTransparency = k
        end)
    end)
end
local function clGhost()
    if ghC then ghC:Disconnect(); ghC=nil end
    for _,x in ipairs(workspace:GetDescendants()) do
        if x.Name=="sGhost" and x:IsA("Highlight") then x:Destroy() end
    end
end

local function mkAtmo()
    local col = FC.atmo
    if not cc then
        cc = Instance.new("ColorCorrectionEffect", L)
        cc.Saturation=0.35; cc.Contrast=0.2; cc.Brightness=0.05
    end
    cc.TintColor = Color3.new(math.min(1,col.R*1.2+0.1),math.min(1,col.G*1.2+0.1),math.min(1,col.B*1.2+0.1))
    if not atmo then
        atmo = Instance.new("Atmosphere", L)
        atmo.Offset=0.2; atmo.Glare=0.3; atmo.Haze=2
    end
    atmo.Density=SZ.atmo; atmo.Color=col; atmo.Decay=LT(col)
end
local function clAtmo()
    if cc then cc:Destroy(); cc=nil end
    if atmo then atmo:Destroy(); atmo=nil end
end

local function mkFB()
    L.Ambient=Color3.new(1,1,1); L.OutdoorAmbient=Color3.new(1,1,1)
    L.Brightness=2; L.GlobalShadows=false
end
local function clFB()
    L.Ambient=Color3.fromRGB(70,70,70)
    L.OutdoorAmbient=Color3.fromRGB(128,128,128)
    L.Brightness=1; L.GlobalShadows=true
end

local function mkSkybox()
    if skyObj then skyObj:Destroy() end
    skyObj = Instance.new("Sky")
    skyObj.SkyboxBk=SID; skyObj.SkyboxDn=SID; skyObj.SkyboxFt=SID
    skyObj.SkyboxLf=SID; skyObj.SkyboxRt=SID; skyObj.SkyboxUp=SID
    skyObj.SunAngularSize=0; skyObj.MoonAngularSize=0
    skyObj.Parent=L
end
local function clSkybox()
    if skyObj then skyObj:Destroy(); skyObj=nil end
end

local function mkBloom()
    if not blm then blm = Instance.new("BloomEffect", L) end
    blm.Intensity=SZ.bloom; blm.Size=32; blm.Threshold=0.85
end
local function clBloom()
    if blm then blm:Destroy(); blm=nil end
end

applyTime = function(preset)
    timePreset = preset
    if preset=="Day" then
        L.ClockTime=14; L.Brightness=2
        L.Ambient=Color3.fromRGB(140,140,140); L.OutdoorAmbient=Color3.fromRGB(160,160,160)
    elseif preset=="Sunset" then
        L.ClockTime=18; L.Brightness=1.6
        L.Ambient=Color3.fromRGB(180,120,90); L.OutdoorAmbient=Color3.fromRGB(220,140,80)
    elseif preset=="Night" then
        L.ClockTime=0; L.Brightness=0.6
        L.Ambient=Color3.fromRGB(40,40,60); L.OutdoorAmbient=Color3.fromRGB(50,50,80)
    elseif preset=="Cyber" then
        L.ClockTime=2; L.Brightness=1.2
        L.Ambient=Color3.fromRGB(90,60,150); L.OutdoorAmbient=Color3.fromRGB(120,80,200)
    end
end
toggleFog = function(v)
    if v then
        L.FogEnd=200; L.FogStart=20; L.FogColor=Color3.fromRGB(60,50,90)
    else
        L.FogEnd=1e6; L.FogStart=0
    end
end

-- ============================================================
-- ESP (uses cachedRole so ghost murderer stays RED)
-- ============================================================
local function applySkinAll()
    local c = LP.Character; if not c then return end
    if fl.skinCol then
        local col = fl.skinCol
        pcall(function()
            for _, p in ipairs(c:GetDescendants()) do
                if p:IsA("BasePart") then p.Color = col end
            end
        end)
    end
    pcall(function()
        local h0 = c:FindFirstChild("Head")
        if h0 then
            for _, d in ipairs(h0:GetChildren()) do
                if d:IsA("SpecialMesh") and (d.Name:sub(1,5) == "sHat_" or d.Name == "sDom") then
                    d:Destroy()
                end
            end
        end
    end)
    local h = c:FindFirstChild("Head")
    if h then
        if fl.headless then
            pcall(function()
                h.Transparency = 1
                local f = h:FindFirstChildOfClass("Decal")
                if f then f.Transparency = 1 end
            end)
        else
            pcall(function()
                h.Transparency = 0
                local f = h:FindFirstChildOfClass("Decal")
                if f then f.Transparency = 0 end
            end)
        end
    end
    pcall(function()
        local rup = c:FindFirstChild("RightUpperLeg")
        if rup then
            local m = rup:FindFirstChild("sKorb")
            if m then m:Destroy() end
            for _, d in ipairs(rup:GetChildren()) do
                if d:IsA("SpecialMesh") then d.Scale = Vector3.new(1,1,1) end
            end
        end
        local rlo = c:FindFirstChild("RightLowerLeg")
        if rlo then rlo.Transparency = 0 end
        local rft = c:FindFirstChild("RightFoot")
        if rft then rft.Transparency = 0 end
    end)
end
local function mkESP()
    if espC then return end
    espC = RS.Heartbeat:Connect(function()
        if not ADDON.loaded then return end
        refreshRoleCache()
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl ~= LP then
                local c = pl.Character
                if c then
                    local r = cachedRole(pl)
                    local col = Color3.fromRGB(60,255,120)
                    if r == "m" then col = Color3.fromRGB(255,60,60)
                    elseif r == "s" then col = Color3.fromRGB(60,140,255) end
                    local h = c:FindFirstChild("sHL")
                    if not h then
                        h = Instance.new("Highlight")
                        h.Name="sHL"; h.OutlineTransparency=0
                        h.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop
                        h.Parent = c
                    end
                    h.FillTransparency = SZ.esp
                    h.FillColor = col
                    h.OutlineColor = col
                elseif fl.espGhost and roleCache[pl.Name] == "Murderer" then
                    -- ghost: РёС‰РµРј РјРѕРґРµР»СЊ РјР°СЂРґРµСЂР° РїРѕ РІСЃРµРјСѓ workspace, Р° РЅРµ С‚РѕР»СЊРєРѕ РІ РєРѕСЂРЅРµ
                    local ghost = workspace:FindFirstChild(pl.Name, true)
                    if ghost and ghost:IsA("Model") and ghost ~= LP.Character then
                        local hasPart = ghost:FindFirstChildWhichIsA("BasePart", true) ~= nil
                        if hasPart then
                            local h = ghost:FindFirstChild("sHL")
                            if not h then
                                h = Instance.new("Highlight")
                                h.Name = "sHL"
                                h.OutlineTransparency = 0
                                h.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                                h.Adornee = ghost
                                h.Parent = workspace
                            else
                                h.Adornee = ghost
                                if h.Parent ~= workspace then h.Parent = workspace end
                            end
                            h.FillTransparency = SZ.esp
                            h.FillColor = Color3.fromRGB(255,60,60)
                            h.OutlineColor = Color3.fromRGB(255,60,60)
                        end
                    end
                end
            end
        end
    end)
end
local function clESP()
    if espC then espC:Disconnect(); espC=nil end
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl.Character then
            for _,x in ipairs(pl.Character:GetDescendants()) do
                if x.Name=="sHL" then x:Destroy() end
            end
        end
    end
    for _, x in ipairs(workspace:GetChildren()) do
        if x:IsA("Highlight") and x.Name == "sHL" then x:Destroy() end
    end
end

enableTracer_fn = function()
    if TRACE.conn then return end
    local m = get_ws_fn()
    local ev = m and m.GunFired
    if typeof(ev)~="Instance" then return end
    TRACE.conn = ev.OnClientEvent:Connect(function(gun,a,b)
        if not fl.tracer then return end
        local c = LP.Character; if not c then return end
        if typeof(gun)~="Instance" or not gun:IsDescendantOf(c) then return end
        local pa, pb = to_pos(a), to_pos(b)
        if not (pa and pb) then
            local cam = workspace.CurrentCamera
            if not cam then return end
            pa = pa or cam.CFrame.Position
            pb = pb or (pa + cam.CFrame.LookVector * 3000)
        end
        local d = pb - pa; local dist = d.Magnitude
        if dist < 0.1 then return end
        local p = Instance.new("Part")
        p.Name="sBeam"; p.Anchored=true
        p.CanCollide=false; p.CanQuery=false; p.CanTouch=false; p.CastShadow=false
        p.Material=Enum.Material.Neon; p.Color=FC.tracer
        p.Size=Vector3.new(SZ.tracer, SZ.tracer, dist)
        p.CFrame = CFrame.lookAt(pa + d/2, pb)
        p.Transparency=0.05; p.Parent=workspace
        local li = Instance.new("PointLight", p)
        li.Color=FC.tracer; li.Brightness=3; li.Range=10
        local ts = 0
        local cn
        cn = RS.Heartbeat:Connect(function(dt)
            ts=ts+dt
            local k=ts/0.6
            if k>=1 or not p.Parent then cn:Disconnect(); if p.Parent then p:Destroy() end; return end
            p.Transparency = 0.05 + k*0.95
            li.Brightness = 3*(1-k)
        end)
    end)
end
disableTracer_fn = function()
    if TRACE.conn then pcall(function() TRACE.conn:Disconnect() end); TRACE.conn=nil end
end

local function mkJump()
    if jumpC then jumpC:Disconnect() end
    local c = LP.Character; if not c then return end
    local h = c:FindFirstChildOfClass("Humanoid"); if not h then return end
    jumpC = h.Jumping:Connect(function(active)
        if not active then return end
        local hrp = HRP(); if not hrp then return end
        local pos = hrp.Position - Vector3.new(0,2.5,0)
        local col = FC.jump
        local function ring()
            local p = Instance.new("Part")
            p.Size=Vector3.new(1,1,1); p.Anchored=true
            p.CanCollide=false; p.CanQuery=false; p.CanTouch=false; p.CastShadow=false
            p.Material=Enum.Material.Neon; p.Color=col; p.Transparency=0.1
            p.CFrame = CFrame.new(pos + Vector3.new(0,0.7,0)) * CFrame.Angles(math.rad(90),0,0)
            local m = Instance.new("SpecialMesh", p)
            m.MeshId="rbxassetid://3270017"
            m.Scale=Vector3.new(SZ.jump,SZ.jump,0.3)
            p.Parent=workspace
            local ts = 0
            local cn
            cn = RS.Heartbeat:Connect(function(dt)
                ts=ts+dt
                local k=ts/0.5
                if k>=1 or not p.Parent then cn:Disconnect(); if p.Parent then p:Destroy() end; return end
                local sz = SZ.jump + k*6
                m.Scale=Vector3.new(sz,sz,0.3*(1-k*0.5))
                p.Transparency = 0.1 + k*0.9
            end)
        end
        if jStyle=="Classic" then ring()
        elseif jStyle=="Double" then ring(); task.delay(0.15, ring)
        elseif jStyle=="Ripple" then
            for i=1,3 do task.delay(i*0.08, ring) end
        elseif jStyle=="Particles" then
            ring()
            local a = Instance.new("Attachment", workspace.Terrain)
            a.WorldPosition = pos
            local pe = Instance.new("ParticleEmitter", a)
            pe.Texture="rbxasset://textures/particles/sparkles_main.dds"
            pe.Rate=0; pe.Lifetime=NumberRange.new(0.6,1)
            pe.Speed=NumberRange.new(6,10); pe.SpreadAngle=Vector2.new(180,180)
            pe.Size=NumberSequence.new({NumberSequenceKeypoint.new(0,0.8),NumberSequenceKeypoint.new(1,0)})
            pe.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0.1),NumberSequenceKeypoint.new(1,1)})
            pe.Color=ColorSequence.new(col, LT(col))
            pe.LightEmission=1; pe.LightInfluence=0; pe.Drag=3
            pe:Emit(35)
            task.delay(1.2, function() if a.Parent then a:Destroy() end end)
        else ring() end
    end)
end
local function clJump()
    if jumpC then jumpC:Disconnect(); jumpC=nil end
end

-- ============================================================
-- KILL LIGHTNING
-- ============================================================
local function killLightning(pos)
    local bm = Instance.new("Part")
    bm.Name="sKl"; bm.Size=Vector3.new(1.5,80,1.5)
    bm.Anchored=true; bm.CanCollide=false; bm.CanQuery=false; bm.CanTouch=false
    bm.CastShadow=false; bm.Material=Enum.Material.Neon
    bm.Color=Color3.fromRGB(200,180,255); bm.Transparency=0.2
    bm.CFrame = CFrame.new(pos + Vector3.new(0,40,0)); bm.Parent=workspace
    local li = Instance.new("PointLight", bm)
    li.Color=Color3.fromRGB(180,150,255); li.Brightness=15; li.Range=40
    local rg = Instance.new("Part")
    rg.Name="sKlR"; rg.Size=Vector3.new(1,1,1)
    rg.Anchored=true; rg.CanCollide=false; rg.CanQuery=false; rg.CanTouch=false
    rg.CastShadow=false; rg.Material=Enum.Material.Neon
    rg.Color=Color3.fromRGB(220,200,255); rg.Transparency=0.3
    rg.CFrame=CFrame.new(pos)
    local rm = Instance.new("SpecialMesh", rg)
    rm.MeshId="rbxassetid://3270017"; rm.Scale=Vector3.new(6,6,0.5)
    rg.Parent=workspace
    local t = 0
    local cn
    cn = RS.Heartbeat:Connect(function(dt)
        t=t+dt
        local k=t/0.7
        if k>=1 or (not bm.Parent and not rg.Parent) then
            cn:Disconnect()
            if bm.Parent then bm:Destroy() end
            if rg.Parent then rg:Destroy() end
            return
        end
        if bm.Parent then
            bm.Transparency = 0.2 + k*0.8
            li.Brightness = 15*(1-k)
        end
        if rg.Parent then
            local rs = 6 + k*10
            rm.Scale = Vector3.new(rs,rs,0.5*(1-k))
            rg.Transparency = 0.3 + k*0.7
        end
    end)
    local att = Instance.new("Attachment", workspace.Terrain)
    att.WorldPosition = pos
    local pe = Instance.new("ParticleEmitter", att)
    pe.Texture="rbxasset://textures/particles/sparkles_main.dds"
    pe.Rate=0; pe.Lifetime=NumberRange.new(0.5,1)
    pe.Speed=NumberRange.new(15,25); pe.SpreadAngle=Vector2.new(180,180)
    pe.Size=NumberSequence.new({NumberSequenceKeypoint.new(0,0.9),NumberSequenceKeypoint.new(1,0)})
    pe.Transparency=NumberSequence.new({NumberSequenceKeypoint.new(0,0.1),NumberSequenceKeypoint.new(1,1)})
    pe.Color=ColorSequence.new(Color3.fromRGB(200,180,255), Color3.new(1,1,1))
    pe.LightEmission=1; pe.LightInfluence=0; pe.Drag=4
    pe:Emit(80)
    task.delay(1.5, function() if att.Parent then att:Destroy() end end)
end

hookKills = function()
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl ~= LP and pl.Character then
            local h2 = pl.Character:FindFirstChildOfClass("Humanoid")
            if h2 and not killHooked[h2] then
                killHooked[h2] = true
                h2.Died:Connect(function()
                    if not killOn then return end
                    task.wait(0.05)
                    local cr = h2:FindFirstChild("creator")
                    if cr and cr.Value == LP then
                        local victim_role = getRole(pl)
                        if victim_role == "m" then
                            local oldPos = h2.Parent and h2.Parent:GetPivot().Position
                            if oldPos then pcall(killLightning, oldPos) end
                        end
                    end
                end)
            end
        end
    end
end

-- ============================================================
-- UI
-- ============================================================
task.spawn(function()
    local t0 = os.clock()
    while not getgenv().UN_BOOT_OK and os.clock() - t0 < 30 do task.wait(0.1) end
    if not getgenv().UN_BOOT_OK then
        local pg0 = LP:FindFirstChild("PlayerGui")
        local bootGone = not pg0 or not pg0:FindFirstChild("UN_BOOT")
        if bootGone then
            getgenv().UN_DEVICE = getgenv().UN_DEVICE or "PC"
            getgenv().UN_LANG = getgenv().UN_LANG or "ru"
        else
            while not getgenv().UN_BOOT_OK do task.wait(0.1) end
        end
    end
    local IS_MOBILE = getgenv().UN_DEVICE == "Mobile"
    local LANG = getgenv().UN_LANG or "ru"
    local LTAB = {
        ru={Combat="Р‘РѕР№",Move="Р”РІРёР¶РµРЅРёРµ",Troll="РўСЂРѕР»Р»СЊ",Visuals="Р’РёР·СѓР°Р»",Jump="РџСЂС‹Р¶РѕРє",World="РњРёСЂ",ESP="Р•РЎРџ",Theme="РўРµРјР°",Extra="Р­РєСЃС‚СЂР°",Btns="РљРЅРѕРїРєРё",
        ["Silent Aim"]="РЎР°Р№Р»РµРЅС‚ РђРёРј",["Silent Predict"]="РџСЂРµРґРёРєС‚",WallCheck="РЎРєРІРѕР·СЊ СЃС‚РµРЅС‹",["Wallbang"]="РЎРєРІРѕР·СЊ СЃС‚РµРЅС‹",["Knife Silent"]="РќРѕР¶ РЎР°Р№Р»РµРЅС‚",["Kill Aura"]="РљРёР»Р» РђСѓСЂР°",["grab gun"]="РђРІС‚Рѕ Р“Р°РЅ",["Bullet Tracer"]="РўСЂР°СЃРµСЂ РџСѓР»СЊ",["Gun Tracer"]="РўСЂР°СЃРµСЂ Р“Р°РЅР°",
        ["Walk:"]="РҐРѕРґСЊР±Р°:",Walkspeed="РЎРєРѕСЂРѕСЃС‚СЊ РҐРѕРґСЊР±С‹",Speed="РЎРєРѕСЂРѕСЃС‚СЊ",["Jump:"]="РџСЂС‹Р¶РѕРє:",JumpPower="РЎРёР»Р° РџСЂС‹Р¶РєР°",Power="РњРѕС‰РЅРѕСЃС‚СЊ",["Infinite Jump"]="Р‘РµСЃРєРѕРЅРµС‡РЅС‹Р№ РџСЂС‹Р¶РѕРє",["Character:"]="РџРµСЂСЃРѕРЅР°Р¶:",HipHeight="Р’С‹СЃРѕС‚Р° РҐРёРїР°",Hip="РҐРёРї",Noclip="РќРѕРєР»РёРї",["Fly:"]="РџРѕР»РµС‚:",Fly="Р¤Р»Р°Р№",FlySpeed="РЎРєРѕСЂРѕСЃС‚СЊ Р¤Р»Р°СЏ",["Spin:"]="РЎРїРёРЅ:",Spin="РЎРїРёРЅ",SpinSpeed="РЎРєРѕСЂРѕСЃС‚СЊ РЎРїРёРЅР°",["Gravity:"]="Р“СЂР°РІРёС‚Р°С†РёСЏ:",["Custom Gravity"]="РЎРІРѕСЏ Р“СЂР°РІРёС‚Р°С†РёСЏ",Gravity="Р“СЂР°РІРёС‚Р°С†РёСЏ",["Reset Move"]="РЎР±СЂРѕСЃ",["Teleport:"]="РўРµР»РµРїРѕСЂС‚:",["TP to Map"]="РўРџ РЅР° РљР°СЂС‚Сѓ",["TP to Lobby"]="РўРџ РІ Р›РѕР±Р±Рё",["Anti Fall (void)"]="РђРЅС‚Рё Р’РѕР№Рґ",["Anti Fling"]="РђРЅС‚Рё Р¤Р»РёРЅРі",["Tracer Settings"]="РќР°СЃС‚СЂРѕР№РєРё РўСЂР°СЃРµСЂР°",
        ["Fling:"]="Р¤Р»РёРЅРі:",["Fling Nearest"]="Р¤Р»РёРЅРі Р‘Р»РёР¶РЅРµРіРѕ",["Fling All"]="Р¤Р»РёРЅРі Р’СЃРµС…",["Fling Murderer"]="Р¤Р»РёРЅРі РњР°СЂРґРµСЂР°",["Fling Sheriff"]="Р¤Р»РёРЅРі РЁРµСЂРёС„Р°",["Anoyances:"]="РџСЂРёРєРѕР»С‹:",["Sit Spam"]="РЎРїР°Рј РЎРёРґРµРЅРёРµРј",["Follow Nearest"]="РЎР»РµРґРёС‚СЊ Р·Р° Р‘Р»РёР¶РЅРёРј",["Stop All Troll"]="РЎС‚РѕРї РўСЂРѕР»Р»СЊ",["Fun:"]="Р¤Р°РЅ:",["TP to Murder"]="РўРџ Рє РњР°СЂРґРµСЂСѓ",["TP to Sheriff"]="РўРџ Рє РЁРµСЂРёС„Сѓ",["Spin Aura (fling around)"]="РЎРїРёРЅ РђСѓСЂР°",["Dizzy (spin+jump)"]="Р”РёР·Р·Рё",["Kill All"]="РЈР±РёС‚СЊ Р’СЃРµС…",
        ["Neon Trail"]="РќРµРѕРЅ РўСЂРµР№Р»",["China Hat"]="РљРёС‚Р°Р№СЃРєР°СЏ РЁР»СЏРїР°",["Body Glow"]="РЎРІРµС‡РµРЅРёРµ РўРµР»Р°",Rainbow="Р Р°РґСѓРіР°",
        ["Aura Particles"]="РђСѓСЂР° Р§Р°СЃС‚РёС†",["Sky Dust"]="РќРµР±РµСЃРЅР°СЏ РџС‹Р»СЊ",["Hearts (Sky): toggle"]="РЎРµСЂРґС†Р° РќРµР±Рѕ",["Snow Aura"]="РЎРЅРµР¶РЅР°СЏ РђСѓСЂР°",["Bounce Particles"]="РџСЂС‹РіР°СЋС‰РёРµ Р§Р°СЃС‚РёС†С‹",["Hearts (Bounce): toggle"]="РЎРµСЂРґС†Р° Р‘Р°СѓРЅСЃ",["Lightning Aura"]="РђСѓСЂР° РњРѕР»РЅРёРё",["Orb Aura"]="РђСѓСЂР° РћСЂР±РѕРІ",["Ghost Trail"]="РџСЂРёР·СЂР°С‡РЅС‹Р№ РЎР»РµРґ",Footsteps="РЎР»РµРґС‹",
        ["Jump Circles"]="РљСЂСѓРіРё РџСЂС‹Р¶РєР°",["Style:"]="РЎС‚РёР»СЊ:",Classic="РљР»Р°СЃСЃРёРєР°",["Double Ring"]="Р”РІРѕР№РЅРѕРµ РљРѕР»СЊС†Рѕ",["Triple Ripple"]="РўСЂРѕР№РЅР°СЏ Р’РѕР»РЅР°",["Particle Burst"]="Р’Р·СЂС‹РІ Р§Р°СЃС‚РёС†",
        ["Cyberpunk Atmo"]="РљРёР±РµСЂРїР°РЅРє РђС‚РјРѕ",FullBright="Р¤СѓР»Р±СЂР°Р№С‚",Skybox="РЎРєР°Р№Р±РѕРєСЃ",Bloom="Р‘Р»СѓРј",["Time Presets:"]="Р’СЂРµРјСЏ:",Day="Р”РµРЅСЊ",Sunset="Р—Р°РєР°С‚",Night="РќРѕС‡СЊ",Cyber="РљРёР±РµСЂ",Fog="РўСѓРјР°РЅ",
        ["Role ESP"]="Р•РЎРџ Р РѕР»РµР№",["Ghost ESP"]="Р•РЎРџ РџСЂРёР·СЂР°РєРѕРІ",["Themes:"]="РўРµРјС‹:",["Config:"]="РљРѕРЅС„РёРі:",Save="РЎРѕС…СЂР°РЅРёС‚СЊ",Load="Р—Р°РіСЂСѓР·РёС‚СЊ",
        ["Kill Lightning (Murderer)"]="РњРѕР»РЅРёСЏ РЈР±РёР№СЃС‚РІР°",Backtrack="Р‘РµРєС‚СЂРµРє",["Utility:"]="РЈС‚РёР»РёС‚С‹:",["Reset colors"]="РЎР±СЂРѕСЃ Р¦РІРµС‚РѕРІ",["Clear particles"]="РћС‡РёСЃС‚РёС‚СЊ Р§Р°СЃС‚РёС†С‹",
        Settings="РќР°СЃС‚СЂРѕР№РєРё",["Size:"]="Р Р°Р·РјРµСЂ:",CLOSE="Р—РђРљР Р«РўР¬",Menu="РњРµРЅСЋ",["SET BIND..."]="РќРђР—РќРђР§РРўР¬ Р‘РРќР”",["CLEAR BIND"]="РЈР‘Р РђРўР¬ Р‘РРќР”",
        trail="РўСЂРµР№Р»",parts="РђСѓСЂР°",skyDust="РџС‹Р»СЊ",snow="РЎРЅРµРі",bounce="Р‘Р°СѓРЅСЃ",lightning="РњРѕР»РЅРёСЏ",orbAura="РћСЂР±С‹",ghost="Р“РѕСЃС‚",foot="РЁР°РіРё",hat="РЁР»СЏРїР°",glow="Р“Р»РѕСѓ",rainbow="Р Р°РґСѓРіР°",jump="РџСЂС‹Р¶РѕРє",atmo="РђС‚РјРѕ",bloom="Р‘Р»СѓРј",esp="Р•РЎРџ",tracer="РўСЂР°СЃРµСЂ",Model="РњРѕРґРµР»СЊ",["Rainbow Body"]="Р Р°РґСѓР¶РЅРѕРµ РўРµР»Рѕ",["Neon Body"]="РќРµРѕРЅРѕРІРѕРµ РўРµР»Рѕ",Invisible="РќРµРІРёРґРёРјРєР°",["Ghost Limbs"]="РџСЂРёР·СЂР°С‡РЅС‹Рµ РєРѕРЅРµС‡РЅРѕСЃС‚Рё",["Big Head"]="Р‘РѕР»СЊС€Р°СЏ РіРѕР»РѕРІР°",Tiny="РњРµР»РєРёР№",["Pumpkin Head"]="РўС‹РєРІРµРЅРЅР°СЏ РіРѕР»РѕРІР°",
        Headless="Р‘РµР· Р“РѕР»РѕРІС‹",["No Hats"]="Р‘РµР· РЁР»СЏРї",Black="Р§С‘СЂРЅС‹Р№",White="Р‘РµР»С‹Р№",Red="РљСЂР°СЃРЅС‹Р№",Cyan="Р¦РёР°РЅ",Pink="Р РѕР·РѕРІС‹Р№",Gold="Р—РѕР»РѕС‚РѕР№",["Ctrl+Click TP"]="РљРўР Р›+РљР»РёРє РўРџ",["Anti AFK"]="РђРЅС‚Рё РђР¤Рљ",["TP to Random"]="РўРџ Рє РЎР»СѓС‡Р°Р№РЅРѕРјСѓ"},
        en={Combat="Combat",Move="Move",Troll="Troll",Visuals="Visuals",Jump="Jump",World="World",ESP="ESP",Theme="Theme",Extra="Extra",Btns="Buttons",
        ["Silent Aim"]="Silent Aim",["Silent Predict"]="Silent Predict",WallCheck="Wall Check",["Wallbang"]="Wallbang",["Knife Silent"]="Knife Silent",["Kill Aura"]="Kill Aura",["grab gun"]="Auto Gun",["Bullet Tracer"]="Bullet Tracer",["Gun Tracer"]="Gun Tracer",
        ["Walk:"]="Walk:",Walkspeed="Walkspeed",Speed="Speed",["Jump:"]="Jump:",JumpPower="JumpPower",Power="Power",["Infinite Jump"]="Infinite Jump",["Character:"]="Character:",HipHeight="HipHeight",Hip="Hip",Noclip="Noclip",["Fly:"]="Fly:",Fly="Fly",FlySpeed="FlySpeed",["Spin:"]="Spin:",Spin="Spin",SpinSpeed="SpinSpeed",["Gravity:"]="Gravity:",["Custom Gravity"]="Custom Gravity",Gravity="Gravity",["Reset Move"]="Reset Move",["Teleport:"]="Teleport:",["TP to Map"]="TP to Map",["TP to Lobby"]="TP to Lobby",["Anti Fall (void)"]="Anti Fall",["Anti Fling"]="Anti Fling",["Tracer Settings"]="Tracer Settings",
        ["Fling:"]="Fling:",["Fling Nearest"]="Fling Nearest",["Fling All"]="Fling All",["Fling Murderer"]="Fling Murderer",["Fling Sheriff"]="Fling Sheriff",["Anoyances:"]="Annoyances:",["Sit Spam"]="Sit Spam",["Follow Nearest"]="Follow Nearest",["Stop All Troll"]="Stop All Troll",["Fun:"]="Fun:",["TP to Murder"]="TP to Murder",["TP to Sheriff"]="TP to Sheriff",["Spin Aura (fling around)"]="Spin Aura",["Dizzy (spin+jump)"]="Dizzy",["Kill All"]="Kill All",
        ["Neon Trail"]="Neon Trail",["China Hat"]="China Hat",["Body Glow"]="Body Glow",Rainbow="Rainbow",
        ["Aura Particles"]="Aura Particles",["Sky Dust"]="Sky Dust",["Hearts (Sky): toggle"]="Hearts (Sky)",["Snow Aura"]="Snow Aura",["Bounce Particles"]="Bounce Particles",["Hearts (Bounce): toggle"]="Hearts (Bounce)",["Lightning Aura"]="Lightning Aura",["Orb Aura"]="Orb Aura",["Ghost Trail"]="Ghost Trail",Footsteps="Footsteps",
        ["Jump Circles"]="Jump Circles",["Style:"]="Style:",Classic="Classic",["Double Ring"]="Double Ring",["Triple Ripple"]="Triple Ripple",["Particle Burst"]="Particle Burst",
        ["Cyberpunk Atmo"]="Cyberpunk Atmo",FullBright="FullBright",Skybox="Skybox",Bloom="Bloom",["Time Presets:"]="Time Presets:",Day="Day",Sunset="Sunset",Night="Night",Cyber="Cyber",Fog="Fog",
        ["Role ESP"]="Role ESP",["Ghost ESP"]="Ghost ESP",["Themes:"]="Themes:",["Config:"]="Config:",Save="Save",Load="Load",
        ["Kill Lightning (Murderer)"]="Kill Lightning",["Backtrack"]="Backtrack",["Utility:"]="Utility:",["Reset colors"]="Reset colors",["Clear particles"]="Clear particles",
        Settings="Settings",["Size:"]="Size:",CLOSE="CLOSE",Menu="Menu",["SET BIND..."]="SET BIND",["CLEAR BIND"]="CLEAR BIND",
        trail="Trail",parts="Aura",skyDust="Dust",snow="Snow",bounce="Bounce",lightning="Lightning",orbAura="Orbs",ghost="Ghost",foot="Steps",hat="Hat",glow="Glow",rainbow="Rainbow",jump="Jump",atmo="Atmo",bloom="Bloom",esp="ESP",tracer="Tracer",Model="Model",["Rainbow Body"]="Rainbow Body",["Neon Body"]="Neon Body",Invisible="Invisible",["Ghost Limbs"]="Ghost Limbs",["Big Head"]="Big Head",Tiny="Tiny",["Pumpkin Head"]="Pumpkin Head",
        Headless="Headless",["No Hats"]="No Hats",Black="Black",White="White",Red="Red",Cyan="Cyan",Pink="Pink",Gold="Gold",["Ctrl+Click TP"]="Ctrl+Click TP",["Anti AFK"]="Anti AFK",["TP to Random"]="TP to Random"},
        uk={Combat="Р‘С–Р№",Move="Р СѓС…",Troll="РўСЂРѕР»СЊ",Visuals="Р’С–Р·СѓР°Р»",Jump="РЎС‚СЂРёР±РѕРє",World="РЎРІС–С‚",ESP="Р•РЎРџ",Theme="РўРµРјР°",Extra="Р•РєСЃС‚СЂР°",Btns="РљРЅРѕРїРєРё",
        ["Silent Aim"]="РЎР°Р№Р»РµРЅС‚ РђС–Рј",["Silent Predict"]="РџСЂРµРґРёРєС‚",WallCheck="РљСЂС–Р·СЊ СЃС‚С–РЅРё",["Wallbang"]="РљСЂС–Р·СЊ СЃС‚С–РЅРё",["Knife Silent"]="РќС–Р¶ РЎР°Р№Р»РµРЅС‚",["Kill Aura"]="РљС–Р» РђСѓСЂР°",["grab gun"]="РђРІС‚Рѕ Р“Р°РЅ",["Bullet Tracer"]="РўСЂР°СЃРµСЂ РљСѓР»СЊ",["Gun Tracer"]="РўСЂР°СЃРµСЂ Р“Р°РЅР°",
        ["Walk:"]="РҐРѕРґСЊР±Р°:",Walkspeed="РЁРІРёРґРєС–СЃС‚СЊ РҐРѕРґСЊР±Рё",Speed="РЁРІРёРґРєС–СЃС‚СЊ",["Jump:"]="РЎС‚СЂРёР±РѕРє:",JumpPower="РЎРёР»Р° РЎС‚СЂРёР±РєР°",Power="РџРѕС‚СѓР¶РЅС–СЃС‚СЊ",["Infinite Jump"]="РќРµСЃРєС–РЅС‡РµРЅРЅРёР№ РЎС‚СЂРёР±РѕРє",["Character:"]="РџРµСЂСЃРѕРЅР°Р¶:",HipHeight="Р’РёСЃРѕС‚Р° РҐС–РїР°",Hip="РҐС–Рї",Noclip="РќРѕРєР»С–Рї",["Fly:"]="РџРѕР»С–С‚:",Fly="Р¤Р»Р°Р№",FlySpeed="РЁРІРёРґРєС–СЃС‚СЊ Р¤Р»Р°СЋ",["Spin:"]="РЎРїС–РЅ:",Spin="РЎРїС–РЅ",SpinSpeed="РЁРІРёРґРєС–СЃС‚СЊ РЎРїС–РЅСѓ",["Gravity:"]="Р“СЂР°РІС–С‚Р°С†С–СЏ:",["Custom Gravity"]="РЎРІРѕСЏ Р“СЂР°РІС–С‚Р°С†С–СЏ",Gravity="Р“СЂР°РІС–С‚Р°С†С–СЏ",["Reset Move"]="РЎРєРёРЅСѓС‚Рё",["Teleport:"]="РўРµР»РµРїРѕСЂС‚:",["TP to Map"]="РўРџ РЅР° РљР°СЂС‚Сѓ",["TP to Lobby"]="РўРџ РІ Р›РѕР±С–",["Anti Fall (void)"]="РђРЅС‚Рё Р’РѕР№Рґ",["Anti Fling"]="РђРЅС‚Рё Р¤Р»С–РЅРі",["Tracer Settings"]="РќР°Р»Р°С€С‚СѓРІР°РЅРЅСЏ РўСЂР°СЃРµСЂР°",
        ["Fling:"]="Р¤Р»С–РЅРі:",["Fling Nearest"]="Р¤Р»С–РЅРі Р‘Р»РёР¶РЅСЊРѕРіРѕ",["Fling All"]="Р¤Р»С–РЅРі Р’СЃС–С…",["Fling Murderer"]="Р¤Р»С–РЅРі РњР°СЂРґРµСЂР°",["Fling Sheriff"]="Р¤Р»С–РЅРі РЁРµСЂРёС„Р°",["Anoyances:"]="РџСЂРёРєРѕР»Рё:",["Sit Spam"]="РЎРїР°Рј РЎРёРґС–РЅРЅСЏРј",["Follow Nearest"]="РЎС‚РµР¶РёС‚Рё",["Stop All Troll"]="РЎС‚РѕРї РўСЂРѕР»СЊ",["Fun:"]="Р¤Р°РЅ:",["TP to Murder"]="РўРџ РґРѕ РњР°СЂРґРµСЂР°",["TP to Sheriff"]="РўРџ РґРѕ РЁРµСЂРёС„Р°",["Spin Aura (fling around)"]="РЎРїС–РЅ РђСѓСЂР°",["Dizzy (spin+jump)"]="Р”С–Р·Р·С–",["Kill All"]="Р’Р±РёС‚Рё Р’СЃС–С…",
        ["Neon Trail"]="РќРµРѕРЅ РўСЂРµР№Р»",["China Hat"]="РљРёС‚Р°Р№СЃСЊРєРёР№ РљР°РїРµР»СЋС…",["Body Glow"]="РЎРІС–С‚С–РЅРЅСЏ РўС–Р»Р°",Rainbow="Р’РµСЃРµР»РєР°",
        ["Aura Particles"]="РђСѓСЂР° Р§Р°СЃС‚РёРЅРѕРє",["Sky Dust"]="РќРµР±РµСЃРЅРёР№ РџРёР»",["Hearts (Sky): toggle"]="РЎРµСЂС†СЏ РќРµР±Рѕ",["Snow Aura"]="РЎРЅС–РіРѕРІР° РђСѓСЂР°",["Bounce Particles"]="РЎС‚СЂРёР±Р°СЋС‡С– Р§Р°СЃС‚РёРЅРєРё",["Hearts (Bounce): toggle"]="РЎРµСЂС†СЏ Р‘Р°СѓРЅСЃ",["Lightning Aura"]="РђСѓСЂР° Р‘Р»РёСЃРєР°РІРєРё",["Orb Aura"]="РђСѓСЂР° РћСЂР±С–РІ",["Ghost Trail"]="РџСЂРёРјР°СЂРЅРёР№ РЎР»С–Рґ",Footsteps="РЎР»С–РґРё",
        ["Jump Circles"]="РљРѕР»Р° РЎС‚СЂРёР±РєР°",["Style:"]="РЎС‚РёР»СЊ:",Classic="РљР»Р°СЃРёРєР°",["Double Ring"]="РџРѕРґРІС–Р№РЅРµ РљС–Р»СЊС†Рµ",["Triple Ripple"]="РџРѕС‚СЂС–Р№РЅР° РҐРІРёР»СЏ",["Particle Burst"]="Р’РёР±СѓС… Р§Р°СЃС‚РёРЅРѕРє",
        ["Cyberpunk Atmo"]="РљС–Р±РµСЂРїР°РЅРє РђС‚РјРѕ",FullBright="Р¤СѓР»Р±СЂР°Р№С‚",Skybox="РЎРєР°Р№Р±РѕРєСЃ",Bloom="Р‘Р»СѓРј",["Time Presets:"]="Р§Р°СЃ:",Day="Р”РµРЅСЊ",Sunset="Р—Р°С…С–Рґ",Night="РќС–С‡",Cyber="РљС–Р±РµСЂ",Fog="РўСѓРјР°РЅ",
        ["Role ESP"]="Р•РЎРџ Р РѕР»РµР№",["Ghost ESP"]="Р•РЎРџ РџСЂРёРІРёРґС–РІ",["Themes:"]="РўРµРјРё:",["Config:"]="РљРѕРЅС„С–Рі:",Save="Р—Р±РµСЂРµРіС‚Рё",Load="Р—Р°РІР°РЅС‚Р°Р¶РёС‚Рё",
        ["Kill Lightning (Murderer)"]="Р‘Р»РёСЃРєР°РІРєР° Р’Р±РёРІСЃС‚РІР°",Backtrack="Р‘РµРєС‚СЂРµРє",["Utility:"]="РЈС‚РёР»С–С‚Рё:",["Reset colors"]="РЎРєРёРЅСѓС‚Рё РљРѕР»СЊРѕСЂРё",["Clear particles"]="РћС‡РёСЃС‚РёС‚Рё Р§Р°СЃС‚РёРЅРєРё",
        Settings="РќР°Р»Р°С€С‚СѓРІР°РЅРЅСЏ",["Size:"]="Р РѕР·РјС–СЂ:",CLOSE="Р—РђРљР РРўР",Menu="РњРµРЅСЋ",["SET BIND..."]="РџР РР—РќРђР§РРўР Р‘Р†РќР”",["CLEAR BIND"]="РџР РР‘Р РђРўР Р‘Р†РќР”",
        trail="РўСЂРµР№Р»",parts="РђСѓСЂР°",skyDust="РџРёР»",snow="РЎРЅС–Рі",bounce="Р‘Р°СѓРЅСЃ",lightning="Р‘Р»РёСЃРєР°РІРєР°",orbAura="РћСЂР±Рё",ghost="Р“РѕСЃС‚",foot="РљСЂРѕРєРё",hat="РљР°РїРµР»СЋС…",glow="Р“Р»РѕСѓ",rainbow="Р’РµСЃРµР»РєР°",jump="РЎС‚СЂРёР±РѕРє",atmo="РђС‚РјРѕ",bloom="Р‘Р»СѓРј",esp="Р•РЎРџ",tracer="РўСЂР°СЃРµСЂ",Model="РњРѕРґРµР»СЊ",["Rainbow Body"]="Р Р°Р№РґСѓР¶РЅРµ РўС–Р»Рѕ",["Neon Body"]="РќРµРѕРЅРѕРІРµ РўС–Р»Рѕ",Invisible="РќРµРІРёРґРёРјРєР°",["Ghost Limbs"]="РџСЂРёРјР°СЂРЅС– РєС–РЅС†С–РІРєРё",["Big Head"]="Р’РµР»РёРєР° РіРѕР»РѕРІР°",Tiny="Р”СЂС–Р±РЅРёР№",["Pumpkin Head"]="Р“Р°СЂР±СѓР·РѕРІР° РіРѕР»РѕРІР°",
        Headless="Р‘РµР· Р“РѕР»РѕРІРё",["No Hats"]="Р‘РµР· РљР°РїРµР»СЋС…Р°",Black="Р§РѕСЂРЅРёР№",White="Р‘С–Р»РёР№",Red="Р§РµСЂРІРѕРЅРёР№",Cyan="Р¦РёР°РЅ",Pink="Р РѕР¶РµРІРёР№",Gold="Р—РѕР»РѕС‚РёР№",["Ctrl+Click TP"]="РљРўР Р›+РљР»С–Рє РўРџ",["Anti AFK"]="РђРЅС‚Рё РђР¤Рљ",["TP to Random"]="РўРџ РґРѕ Р’РёРїР°РґРєРѕРІРѕРіРѕ"},
    }
    local function T(s) local t=LTAB[LANG] or LTAB.ru return t[s] or s end
    getgenv().UN_T = T
    getgenv().UN_MOBILE = IS_MOBILE
    local function pressFx(b)
        pcall(function()
            b.AutoButtonColor=false
            b.MouseButton1Down:Connect(function() pcall(function() b.BackgroundTransparency=0.35 end) end)
            b.MouseButton1Up:Connect(function() pcall(function() b.BackgroundTransparency=0 end) end)
            b.MouseLeave:Connect(function() pcall(function() b.BackgroundTransparency=0 end) end)
        end)
    end
    local PlayerGui = LP:WaitForChild("PlayerGui")
    local old = PlayerGui:FindFirstChild("UN_ADDON_GUI")
    if old then pcall(function() old:Destroy() end) end

    local sg = Instance.new("ScreenGui")
    sg.Name = "UN_ADDON_GUI"
    sg.ResetOnSpawn = false
    sg.DisplayOrder = 999999
    sg.Parent = PlayerGui

    local setP
    local function closeSet()
        if setP then setP:Destroy(); setP=nil end
    end

    local main = Instance.new("Frame", sg)
    if IS_MOBILE then
        main.Size=UDim2.new(0.94,0,0.82,0)
        main.Position=UDim2.new(0.03,0,0.09,0)
    else
        main.Size=UDim2.new(0,620,0,470)
        main.Position=UDim2.new(0.5,-310,0.5,-235)
    end
    main.BackgroundColor3=BG; main.BorderSizePixel=0
    main.Active=true; main.Visible=true
    Instance.new("UICorner", main).CornerRadius = UDim.new(0,14)
    local mst = Instance.new("UIStroke", main); mst.Color=A; mst.Thickness=2
    local mg = Instance.new("UIGradient", main)
    mg.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, BG), ColorSequenceKeypoint.new(1, Color3.fromRGB(4,7,14))})
    mg.Rotation = 75
    local mainScale=Instance.new("UIScale",main); mainScale.Scale=0.92
    pcall(function() Tween:Create(mainScale, TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale=1}):Play() end)

    local tt = Instance.new("TextLabel", main)
    tt.Size=UDim2.new(1,0,0,44); tt.BackgroundColor3=BG2; tt.BorderSizePixel=0
    tt.Text=""
    Instance.new("UICorner", tt).CornerRadius = UDim.new(0,14)
    local ttg = Instance.new("UIGradient", tt)
    ttg.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, BG2), ColorSequenceKeypoint.new(1, BG)})
    ttg.Rotation = 90
    local logo = Instance.new("TextLabel", tt)
    logo.Size=UDim2.new(0,28,0,28); logo.Position=UDim2.new(0,10,0,8)
    logo.BackgroundColor3=A; logo.BorderSizePixel=0
    logo.Text="в…"; logo.TextColor3=BG; logo.Font=Enum.Font.GothamBlack; logo.TextSize=18
    Instance.new("UICorner", logo).CornerRadius = UDim.new(0,9)
    local nm = Instance.new("TextLabel", tt)
    nm.Size=UDim2.new(0,220,0,22); nm.Position=UDim2.new(0,46,0,5)
    nm.BackgroundTransparency=1; nm.Text="StarWare"; nm.TextColor3=Color3.new(1,1,1)
    nm.Font=Enum.Font.GothamBlack; nm.TextSize=16; nm.TextXAlignment=Enum.TextXAlignment.Left
    local ver = Instance.new("TextLabel", tt)
    ver.Size=UDim2.new(0,220,0,13); ver.Position=UDim2.new(0,47,0,26)
    ver.BackgroundTransparency=1; ver.Text="v2.0 вЂў MM2"; ver.TextColor3=A2
    ver.Font=Enum.Font.GothamMedium; ver.TextSize=10; ver.TextXAlignment=Enum.TextXAlignment.Left
    local accent = Instance.new("Frame", main)
    accent.Size=UDim2.new(1,-24,0,2); accent.Position=UDim2.new(0,12,0,46)
    accent.BackgroundColor3=A; accent.BorderSizePixel=0

    local closeBtn = Instance.new("TextButton", main)
    closeBtn.Size=UDim2.new(0,24,0,24); closeBtn.Position=UDim2.new(1,-32,0,10)
    closeBtn.BackgroundColor3=Color3.fromRGB(80,25,35); closeBtn.Text="X"
    closeBtn.TextColor3=Color3.fromRGB(255,180,180)
    closeBtn.Font=Enum.Font.GothamBold; closeBtn.TextSize=12; closeBtn.BorderSizePixel=0
    Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(1,0)
    pressFx(closeBtn)
    closeBtn.MouseButton1Click:Connect(function()
        pcall(function() Tween:Create(mainScale, TweenInfo.new(0.14, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Scale=0.92}):Play() end)
        task.delay(0.14, function() main.Visible=false end)
    end)

    local sideW = IS_MOBILE and 104 or 138
    local sb = Instance.new("Frame", main)
    sb.Size=UDim2.new(0,sideW,1,-64); sb.Position=UDim2.new(0,8,0,56)
    sb.BackgroundColor3=BG2; sb.BorderSizePixel=0
    Instance.new("UICorner", sb).CornerRadius = UDim.new(0,12)
    local sbl = Instance.new("UIListLayout", sb)
    sbl.FillDirection = Enum.FillDirection.Vertical
    sbl.Padding = UDim.new(0,4)
    sbl.HorizontalAlignment = Enum.HorizontalAlignment.Center
    sbl.SortOrder = Enum.SortOrder.LayoutOrder
    local sbp = Instance.new("UIPadding", sb)
    sbp.PaddingTop = UDim.new(0,8); sbp.PaddingBottom = UDim.new(0,8)
    local selBar = Instance.new("Frame", sb)
    selBar.Size=UDim2.new(0,3,0,26); selBar.BackgroundColor3=A; selBar.BorderSizePixel=0
    selBar.Visible=false
    Instance.new("UICorner", selBar).CornerRadius = UDim.new(1,0)
    local function moveBar(btn)
        local y = btn.AbsolutePosition.Y - sb.AbsolutePosition.Y
        selBar.Visible=true
        selBar.Size=UDim2.new(0,3,0,btn.AbsoluteSize.Y-8)
        pcall(function() Tween:Create(selBar, TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position=UDim2.new(0,5,0,y+4)}):Play() end)
    end

    local hold = Instance.new("Frame", main)
    hold.Size=UDim2.new(1,-sideW-24,1,-64); hold.Position=UDim2.new(0,sideW+16,0,56)
    hold.BackgroundTransparency=1; hold.BorderSizePixel=0; hold.ClipsDescendants=true
    local holdScale=Instance.new("UIScale",hold); holdScale.Scale=1

    local veil = Instance.new("Frame", hold)
    veil.Size=UDim2.new(1,0,1,0); veil.BackgroundColor3=A
    veil.BackgroundTransparency=1; veil.BorderSizePixel=0; veil.ZIndex=50
    Instance.new("UICorner", veil).CornerRadius = UDim.new(0,12)
    local function flash()
        pcall(function()
            veil.BackgroundTransparency=0.8
            Tween:Create(veil, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency=1}):Play()
        end)
    end

    if not IS_MOBILE then
        local function orb(x, y, s, c)
            local o = Instance.new("Frame", main)
            o.Size=UDim2.new(0,s,0,s); o.Position=UDim2.new(0,x,0,y)
            o.BackgroundColor3=c; o.BackgroundTransparency=0.85; o.BorderSizePixel=0; o.ZIndex=0
            Instance.new("UICorner", o).CornerRadius = UDim.new(1,0)
            task.spawn(function()
                while o.Parent do
                    pcall(function() Tween:Create(o, TweenInfo.new(3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Position=o.Position+UDim2.new(0,26,0,-20)}):Play() end)
                    task.wait(3.1)
                    if not o.Parent then break end
                    pcall(function() Tween:Create(o, TweenInfo.new(3, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Position=o.Position-UDim2.new(0,26,0,-20)}):Play() end)
                    task.wait(3.1)
                end
            end)
        end
        orb(30, 380, 150, A)
        orb(430, 70, 110, Color3.fromRGB(150,80,255))
    end
    task.spawn(function()
        while main.Parent do
            pcall(function() Tween:Create(accent, TweenInfo.new(1.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency=0.5}):Play() end)
            task.wait(1.3)
            if not main.Parent then break end
            pcall(function() Tween:Create(accent, TweenInfo.new(1.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {BackgroundTransparency=0}):Play() end)
            task.wait(1.3)
        end
    end)

    local openBtn = Instance.new("TextButton", sg)
    openBtn.Size=UDim2.new(0,100,0,44)
    openBtn.Position=UDim2.new(1,-116,0,20)
    openBtn.BackgroundColor3=A
    openBtn.Text="в… "..T("Menu"); openBtn.TextColor3=Color3.new(1,1,1)
    openBtn.Font=Enum.Font.GothamBlack; openBtn.TextSize=14
    openBtn.BorderSizePixel=0; openBtn.AutoButtonColor=false
    Instance.new("UICorner", openBtn).CornerRadius = UDim.new(0,12)
    local obStroke = Instance.new("UIStroke", openBtn)
    obStroke.Color=Color3.fromRGB(255,180,255); obStroke.Thickness=2
    pressFx(openBtn)
    openBtn.MouseButton1Click:Connect(function()
        main.Visible = not main.Visible
        if main.Visible then
            mainScale.Scale=0.92
            pcall(function() Tween:Create(mainScale, TweenInfo.new(0.22, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale=1}):Play() end)
        end
    end)

    local tabs, pages = {}, {}
    local function paintTab(x, sel)
        if sel then
            x.b.BackgroundColor3=BG3; x.b.TextColor3=A
        else
            pcall(function() Tween:Create(x.b, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG2}):Play() end)
            x.b.TextColor3=Color3.fromRGB(170,180,205)
        end
    end
    local function aTab(name, w)
        local i = #tabs+1
        local b = Instance.new("TextButton", sb)
        b.Size=UDim2.new(1,-12,0,IS_MOBILE and 30 or 34)
        b.BackgroundColor3=BG2; b.TextColor3=Color3.fromRGB(170,180,205)
        b.Font=Enum.Font.GothamBold; b.TextSize=IS_MOBILE and 10 or 12
        b.Text=name; b.BorderSizePixel=0; b.AutoButtonColor=false
        b.TextXAlignment=Enum.TextXAlignment.Left
        b.TextTruncate=Enum.TextTruncate.AtEnd
        Instance.new("UICorner", b).CornerRadius = UDim.new(0,8)
        local pad = Instance.new("UIPadding", b); pad.PaddingLeft=UDim.new(0,10)
        pressFx(b)
        local t = {b=b, sel=false}
        b.MouseEnter:Connect(function()
            pcall(function() Tween:Create(b, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG3}):Play() end)
        end)
        b.MouseLeave:Connect(function() paintTab(t, t.sel) end)
        tabs[#tabs+1]=t
        local p = Instance.new("ScrollingFrame", hold)
        p.Size=UDim2.new(1,0,1,0); p.BackgroundTransparency=1; p.BorderSizePixel=0
        p.ScrollBarThickness=2; p.ScrollBarImageColor3=A
        p.CanvasSize=UDim2.new(0,0,0,0)
        p.AutomaticCanvasSize = Enum.AutomaticSize.Y
        p.Visible=false
        local l = Instance.new("UIListLayout", p); l.Padding = UDim.new(0,5)
        pages[#pages+1]=p
        b.MouseButton1Click:Connect(function()
            for k,x in ipairs(tabs) do x.sel=false paintTab(x, false) end
            for k,x in ipairs(pages) do x.Visible = (k==i) end
            t.sel=true
            paintTab(t, true)
            moveBar(b)
            flash()
            holdScale.Scale=0.97
            pcall(function() Tween:Create(holdScale, TweenInfo.new(0.16, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Scale=1}):Play() end)
        end)
        if i==1 then
            t.sel=true
            paintTab(t, true)
            p.Visible=true
            task.delay(0.15, function() pcall(function() moveBar(b) end) end)
        end
        return p
    end

    local cps = {
        Color3.fromRGB(160,80,255), Color3.fromRGB(255,80,220), Color3.fromRGB(60,220,255),
        Color3.fromRGB(255,60,60), Color3.fromRGB(60,255,120), Color3.fromRGB(255,220,60),
        Color3.fromRGB(255,255,255), Color3.fromRGB(255,140,40), Color3.fromRGB(120,255,220),
        Color3.fromRGB(255,180,220), Color3.fromRGB(100,100,255), Color3.fromRGB(220,220,220),
        Color3.fromRGB(255,0,128), Color3.fromRGB(0,255,200), Color3.fromRGB(255,120,0), Color3.fromRGB(120,255,60),
    }

    local function showSet(key, refreshFn)
        closeSet()
        local p = Instance.new("Frame", sg)
        p.Size=UDim2.new(0,240,0,260)
        local mx = UIS:GetMouseLocation()
        local vp = workspace.CurrentCamera.ViewportSize
        local px,py = mx.X, mx.Y
        if px+240 > vp.X then px = vp.X - 250 end
        if py+260 > vp.Y then py = vp.Y - 270 end
        p.Position=UDim2.new(0,px,0,py)
        p.BackgroundColor3=BG; p.BorderSizePixel=0; p.ZIndex=500
        Instance.new("UICorner", p).CornerRadius = UDim.new(0,8)
        local ps = Instance.new("UIStroke", p); ps.Color=A; ps.Thickness=1.5
        local hd = Instance.new("TextLabel", p)
        hd.Size=UDim2.new(1,0,0,24); hd.BackgroundColor3=BG2; hd.BorderSizePixel=0
        hd.Text="  "..T(key).." вЂў "..T("Settings"); hd.TextColor3=A
        hd.Font=Enum.Font.GothamBold; hd.TextSize=11
        hd.TextXAlignment=Enum.TextXAlignment.Left; hd.ZIndex=501
        Instance.new("UICorner", hd).CornerRadius = UDim.new(0,8)
        local xb = Instance.new("TextButton", p)
        xb.Size=UDim2.new(0,20,0,20); xb.Position=UDim2.new(1,-24,0,2)
        xb.BackgroundColor3=Color3.fromRGB(80,30,50); xb.Text="X"
        xb.TextColor3=Color3.fromRGB(255,180,180)
        xb.Font=Enum.Font.GothamBold; xb.TextSize=11; xb.BorderSizePixel=0; xb.ZIndex=502
        Instance.new("UICorner", xb).CornerRadius = UDim.new(0,4)
        xb.MouseButton1Click:Connect(closeSet)
        local bd = Instance.new("Frame", p)
        bd.Size=UDim2.new(1,-10,1,-64); bd.Position=UDim2.new(0,5,0,28)
        bd.BackgroundTransparency=1; bd.ZIndex=501
        local bl = Instance.new("UIListLayout", bd); bl.Padding = UDim.new(0,5)
        local cw = Instance.new("Frame", bd)
        cw.Size=UDim2.new(1,0,0,50); cw.BackgroundColor3=BG3; cw.BorderSizePixel=0; cw.ZIndex=502
        Instance.new("UICorner", cw).CornerRadius = UDim.new(0,4)
        local cwg = Instance.new("UIGridLayout", cw)
        cwg.CellSize=UDim2.new(0,22,0,22); cwg.CellPadding=UDim2.new(0,3,0,3)
        cwg.FillDirectionMaxCells=9
        for _, c in ipairs(cps) do
            local cb = Instance.new("TextButton", cw)
            cb.BackgroundColor3=c; cb.Text=""; cb.BorderSizePixel=0; cb.ZIndex=503
            Instance.new("UICorner", cb).CornerRadius = UDim.new(0,3)
            cb.MouseButton1Click:Connect(function()
                FC[key] = c
                pcall(refreshFn)
            end)
        end
        if SZ[key] then
            local sr = Instance.new("Frame", bd)
            sr.Size=UDim2.new(1,0,0,26); sr.BackgroundColor3=BG3; sr.BorderSizePixel=0; sr.ZIndex=502
            Instance.new("UICorner", sr).CornerRadius = UDim.new(0,4)
            local sl = Instance.new("TextLabel", sr)
            sl.Size=UDim2.new(0,120,1,0); sl.Position=UDim2.new(0,8,0,0)
            sl.BackgroundTransparency=1
            sl.Text="Size: "..string.format("%.1f", SZ[key])
            sl.TextColor3=A2; sl.Font=Enum.Font.GothamMedium; sl.TextSize=11
            sl.TextXAlignment=Enum.TextXAlignment.Left; sl.ZIndex=503
            local mn = Instance.new("TextButton", sr)
            mn.Size=UDim2.new(0,28,1,-6); mn.Position=UDim2.new(1,-70,0,3)
            mn.BackgroundColor3=Color3.fromRGB(45,32,70); mn.Text="-"
            mn.TextColor3=Color3.new(1,1,1); mn.Font=Enum.Font.GothamBold
            mn.TextSize=14; mn.BorderSizePixel=0; mn.ZIndex=503
            Instance.new("UICorner", mn).CornerRadius = UDim.new(0,4)
            local pl2 = Instance.new("TextButton", sr)
            pl2.Size=UDim2.new(0,28,1,-6); pl2.Position=UDim2.new(1,-36,0,3)
            pl2.BackgroundColor3=Color3.fromRGB(45,32,70); pl2.Text="+"
            pl2.TextColor3=Color3.new(1,1,1); pl2.Font=Enum.Font.GothamBold
            pl2.TextSize=14; pl2.BorderSizePixel=0; pl2.ZIndex=503
            Instance.new("UICorner", pl2).CornerRadius = UDim.new(0,4)
            mn.MouseButton1Click:Connect(function()
                SZ[key] = math.max(0.05, SZ[key]-0.4)
                sl.Text="Size: "..string.format("%.1f", SZ[key])
                pcall(refreshFn)
            end)
            pl2.MouseButton1Click:Connect(function()
                SZ[key] = math.min(20, SZ[key]+0.4)
                sl.Text="Size: "..string.format("%.1f", SZ[key])
                pcall(refreshFn)
            end)
        end
        local cb2 = Instance.new("TextButton", p)
        cb2.Size=UDim2.new(1,-10,0,26); cb2.Position=UDim2.new(0,5,1,-31)
        cb2.BackgroundColor3=Color3.fromRGB(100,35,55); cb2.Text="CLOSE"
        cb2.TextColor3=Color3.fromRGB(255,200,200)
        cb2.Font=Enum.Font.GothamBold; cb2.TextSize=11; cb2.BorderSizePixel=0; cb2.ZIndex=502
        Instance.new("UICorner", cb2).CornerRadius = UDim.new(0,6)
        cb2.MouseButton1Click:Connect(closeSet)
        setP = p
    end

    local function showNum(cfg)
        closeSet()
        local p = Instance.new("Frame", sg)
        p.Size=UDim2.new(0,240,0,110)
        local mx = UIS:GetMouseLocation()
        local vp = workspace.CurrentCamera.ViewportSize
        local px,py = mx.X, mx.Y
        if px+240 > vp.X then px = vp.X - 250 end
        if py+110 > vp.Y then py = vp.Y - 120 end
        p.Position=UDim2.new(0,px,0,py)
        p.BackgroundColor3=BG; p.BorderSizePixel=0; p.ZIndex=500
        Instance.new("UICorner", p).CornerRadius = UDim.new(0,8)
        local ps = Instance.new("UIStroke", p); ps.Color=A; ps.Thickness=1.5
        local hd = Instance.new("TextLabel", p)
        hd.Size=UDim2.new(1,0,0,24); hd.BackgroundColor3=BG2; hd.BorderSizePixel=0
        hd.Text="  "..T(cfg.t).." вЂў "..T("Settings"); hd.TextColor3=A
        hd.Font=Enum.Font.GothamBold; hd.TextSize=11
        hd.TextXAlignment=Enum.TextXAlignment.Left; hd.ZIndex=501
        Instance.new("UICorner", hd).CornerRadius = UDim.new(0,8)
        local xb = Instance.new("TextButton", p)
        xb.Size=UDim2.new(0,20,0,20); xb.Position=UDim2.new(1,-24,0,2)
        xb.BackgroundColor3=Color3.fromRGB(80,30,50); xb.Text="X"
        xb.TextColor3=Color3.fromRGB(255,180,180)
        xb.Font=Enum.Font.GothamBold; xb.TextSize=11; xb.BorderSizePixel=0; xb.ZIndex=502
        Instance.new("UICorner", xb).CornerRadius = UDim.new(0,4)
        xb.MouseButton1Click:Connect(closeSet)
        local sr = Instance.new("Frame", p)
        sr.Size=UDim2.new(1,-10,0,30); sr.Position=UDim2.new(0,5,0,32)
        sr.BackgroundColor3=BG3; sr.BorderSizePixel=0; sr.ZIndex=502
        Instance.new("UICorner", sr).CornerRadius = UDim.new(0,4)
        local sl = Instance.new("TextLabel", sr)
        sl.Size=UDim2.new(1,-76,1,0); sl.Position=UDim2.new(0,8,0,0)
        sl.BackgroundTransparency=1
        sl.TextColor3=A2; sl.Font=Enum.Font.GothamMedium; sl.TextSize=12
        sl.TextXAlignment=Enum.TextXAlignment.Left; sl.ZIndex=503
        local function upd()
            sl.Text = T(cfg.t)..": "..tostring(cfg.get())
        end
        upd()
        local mn = Instance.new("TextButton", sr)
        mn.Size=UDim2.new(0,30,1,-6); mn.Position=UDim2.new(1,-66,0,3)
        mn.BackgroundColor3=Color3.fromRGB(45,32,70); mn.Text="-"
        mn.TextColor3=Color3.new(1,1,1); mn.Font=Enum.Font.GothamBold
        mn.TextSize=14; mn.BorderSizePixel=0; mn.ZIndex=503
        Instance.new("UICorner", mn).CornerRadius = UDim.new(0,4)
        local pl2 = Instance.new("TextButton", sr)
        pl2.Size=UDim2.new(0,30,1,-6); pl2.Position=UDim2.new(1,-32,0,3)
        pl2.BackgroundColor3=Color3.fromRGB(45,32,70); pl2.Text="+"
        pl2.TextColor3=Color3.new(1,1,1); pl2.Font=Enum.Font.GothamBold
        pl2.TextSize=14; pl2.BorderSizePixel=0; pl2.ZIndex=503
        Instance.new("UICorner", pl2).CornerRadius = UDim.new(0,4)
        mn.MouseButton1Click:Connect(function()
            cfg.set(math.max(cfg.min, cfg.get()-cfg.step)); upd()
            if cfg.apply then pcall(cfg.apply) end
        end)
        pl2.MouseButton1Click:Connect(function()
            cfg.set(math.min(cfg.max, cfg.get()+cfg.step)); upd()
            if cfg.apply then pcall(cfg.apply) end
        end)
        local cb2 = Instance.new("TextButton", p)
        cb2.Size=UDim2.new(1,-10,0,26); cb2.Position=UDim2.new(0,5,1,-31)
        cb2.BackgroundColor3=Color3.fromRGB(100,35,55); cb2.Text="CLOSE"
        cb2.TextColor3=Color3.fromRGB(255,200,200)
        cb2.Font=Enum.Font.GothamBold; cb2.TextSize=11; cb2.BorderSizePixel=0; cb2.ZIndex=502
        Instance.new("UICorner", cb2).CornerRadius = UDim.new(0,6)
        cb2.MouseButton1Click:Connect(closeSet)
        setP = p
    end

    local bindOverlay = Instance.new("TextLabel", sg)
    bindOverlay.Name = "BindOverlay"
    bindOverlay.Size = UDim2.new(1,0,1,0)
    bindOverlay.BackgroundColor3 = Color3.fromRGB(0,0,0)
    bindOverlay.BackgroundTransparency = 0.5
    bindOverlay.TextColor3 = Color3.new(1,1,1)
    bindOverlay.Font = Enum.Font.GothamBold
    bindOverlay.TextSize = 24
    bindOverlay.Text = "press any key ..."
    bindOverlay.Visible = false
    bindOverlay.ZIndex = 2000
    bindOverlay.Parent = sg

    task.spawn(function()
        while ADDON.loaded do
            if bindPicking then
                bindOverlay.Visible = true
                bindOverlay.Text = bindPicking.name.."\n"..(LANG=="ru" and "РЅР°Р¶РјРё РєР»Р°РІРёС€Сѓ (ESC=РѕС‚РјРµРЅР°)" or LANG=="uk" and "РЅР°С‚РёСЃРЅРё РєР»Р°РІС–С€Сѓ (ESC=СЃРєР°СЃСѓРІР°С‚Рё)" or "press any key (ESC=cancel)")
            else
                bindOverlay.Visible = false
            end
            task.wait(0.05)
        end
    end)

    local bindMenu = nil
    local function closeBindMenu()
        if bindMenu then pcall(function() bindMenu:Destroy() end) bindMenu = nil end
    end
    local function openBindMenu(bindName, displayName)
        closeBindMenu()
        local bd = BINDS[bindName]
        local cur = (bd and bd.input) and inputName(bd.input) or "NONE"
        local m = UIS:GetMouseLocation()
        local vp = workspace.CurrentCamera.ViewportSize
        local px, py = m.X + 10, m.Y + 10
        if px + 220 > vp.X then px = vp.X - 230 end
        if py + 140 > vp.Y then py = vp.Y - 150 end
        local f = Instance.new("Frame", sg)
        f.Size = UDim2.new(0,210,0,130)
        f.Position = UDim2.new(0,px,0,py)
        f.BackgroundColor3 = BG
        f.BorderSizePixel = 0
        f.ZIndex = 1500
        Instance.new("UICorner", f).CornerRadius = UDim.new(0,8)
        local st = Instance.new("UIStroke", f) st.Color = A st.Thickness = 1.5
        local hd = Instance.new("TextLabel", f)
        hd.Size = UDim2.new(1,0,0,24) hd.BackgroundColor3 = BG2 hd.BorderSizePixel = 0
        hd.Text = "  "..tostring(displayName).." ["..cur.."]" hd.TextColor3 = A
        hd.Font = Enum.Font.GothamBold hd.TextSize = 11 hd.TextXAlignment = Enum.TextXAlignment.Left hd.ZIndex = 1501
        Instance.new("UICorner", hd).CornerRadius = UDim.new(0,8)
        local function mbtn(txt, y, cb)
            local b = Instance.new("TextButton", f)
            b.Size = UDim2.new(1,-10,0,26) b.Position = UDim2.new(0,5,0,y)
            b.BackgroundColor3 = BG3 b.Text = txt b.TextColor3 = Color3.new(1,1,1)
            b.Font = Enum.Font.GothamBold b.TextSize = 11 b.BorderSizePixel = 0 b.ZIndex = 1501
            Instance.new("UICorner", b).CornerRadius = UDim.new(0,6)
            pressFx(b)
            b.MouseButton1Click:Connect(function() pcall(cb) end)
            return b
        end
        mbtn(T("SET BIND..."), 30, function() closeBindMenu() openBindPicker(bindName, displayName) end)
        mbtn(T("CLEAR BIND"), 60, function()
            if BINDS[bindName] then BINDS[bindName].input = nil saveBinds() end
            if BINDS[bindName] and BINDS[bindName].onUpdate then pcall(BINDS[bindName].onUpdate) end
            closeBindMenu()
        end)
        mbtn(T("CLOSE"), 90, closeBindMenu)
        bindMenu = f
        task.delay(6, function() if bindMenu == f then closeBindMenu() end end)
    end

    local function mkToggle(parent, name, getter, setter, makeFn, clFn, key, bindKey, numCfg)
        local bindName = bindKey or name
        name = T(name)
        local row = Instance.new("TextButton", parent)
        if IS_MOBILE then row.Size=UDim2.new(1,-4,0,44)
        else row.Size=UDim2.new(1,-4,0,34) end
        row.BackgroundColor3=BG3; row.Text=""; row.BorderSizePixel=0; row.AutoButtonColor=false
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,10)
        local lbl = Instance.new("TextLabel", row)
        lbl.Size=UDim2.new(1,-64,1,0); lbl.Position=UDim2.new(0,12,0,0)
        lbl.BackgroundTransparency=1; lbl.TextColor3=Color3.new(1,1,1)
        lbl.Font=Enum.Font.GothamMedium; lbl.TextSize=IS_MOBILE and 14 or 12
        lbl.TextXAlignment=Enum.TextXAlignment.Left
        lbl.TextTruncate=Enum.TextTruncate.AtEnd
        local pill = Instance.new("Frame", row)
        pill.Size=UDim2.new(0,40,0,20); pill.Position=UDim2.new(1,-48,0.5,-10)
        pill.BackgroundColor3=Color3.fromRGB(70,70,90); pill.BorderSizePixel=0
        Instance.new("UICorner", pill).CornerRadius = UDim.new(1,0)
        local knob = Instance.new("Frame", pill)
        knob.Size=UDim2.new(0,14,0,14); knob.Position=UDim2.new(0,3,0.5,-7)
        knob.BackgroundColor3=Color3.new(1,1,1); knob.BorderSizePixel=0
        Instance.new("UICorner", knob).CornerRadius = UDim.new(1,0)
        pressFx(row)
        row.MouseEnter:Connect(function()
            pcall(function() Tween:Create(row, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG2}):Play() end)
        end)
        row.MouseLeave:Connect(function()
            pcall(function() Tween:Create(row, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG3}):Play() end)
        end)

        local function upd()
            local on = getter()
            pcall(function()
                Tween:Create(pill, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3 = on and A or Color3.fromRGB(70,70,90)}):Play()
                Tween:Create(knob, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = on and UDim2.new(1,-17,0.5,-7) or UDim2.new(0,3,0.5,-7)}):Play()
            end)
            local bindStr = ""
            local bd = BINDS[bindName]
            if bd and bd.input then bindStr = "  ["..inputName(bd.input).."]" end
            lbl.Text = name..bindStr
        end
        upd()
        row.MouseButton1Click:Connect(function()
            setter(not getter())
            upd()
            notif(name, getter())
        end)
        if numCfg then
            row.MouseButton2Click:Connect(function()
                showNum(numCfg)
            end)
        elseif key and makeFn and clFn and FC[key] then
            row.MouseButton2Click:Connect(function()
                showSet(key, function()
                    if getter() then clFn(); makeFn() end
                end)
            end)
        end
        registerBind(bindName, name, getter, setter)
        BINDS[bindName].onUpdate = upd
        row.InputBegan:Connect(function(input, gp)
            if not IS_MOBILE and input.UserInputType == Enum.UserInputType.MouseButton3 then
                openBindMenu(bindName, name)
            end
        end)
        return row
    end

    local function mkBtn(parent, name, cb)
        name = T(name)
        local b = Instance.new("TextButton", parent)
        b.Size=UDim2.new(1,-4,0,IS_MOBILE and 36 or 30)
        b.BackgroundColor3=BG3
        b.TextColor3=A2; b.Font=Enum.Font.GothamBold; b.TextSize=IS_MOBILE and 13 or 12
        b.Text=name; b.BorderSizePixel=0; b.AutoButtonColor=false
        Instance.new("UICorner", b).CornerRadius = UDim.new(0,10)
        local edge = Instance.new("Frame", b)
        edge.Size=UDim2.new(1,-16,0,2); edge.Position=UDim2.new(0,8,0,2)
        edge.BackgroundColor3=A; edge.BackgroundTransparency=0.6; edge.BorderSizePixel=0
        pressFx(b)
        b.MouseEnter:Connect(function()
            pcall(function() Tween:Create(b, TweenInfo.new(0.12, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG2}):Play() end)
        end)
        b.MouseLeave:Connect(function()
            pcall(function() Tween:Create(b, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundColor3=BG3}):Play() end)
        end)
        b.MouseButton1Click:Connect(function() pcall(cb) end)
        return b
    end

    local function mkLbl(parent, name)
        name = T(name)
        local b = Instance.new("TextLabel", parent)
        b.Size=UDim2.new(1,-4,0,22)
        b.BackgroundTransparency=1; b.TextColor3=A
        b.Font=Enum.Font.GothamBlack; b.TextSize=11
        b.Text="вЂ”  "..name; b.TextXAlignment=Enum.TextXAlignment.Left
        local line = Instance.new("Frame", b)
        line.Size=UDim2.new(1,0,0,1); line.Position=UDim2.new(0,0,1,-1)
        line.BackgroundColor3=A; line.BackgroundTransparency=0.55; line.BorderSizePixel=0
        return b
    end

    local function mkNum(parent, label, getter, setter, step, minv, maxv, suffix, onChange)
        label = T(label)
        local row = Instance.new("Frame", parent)
        row.Size=UDim2.new(1,-4,0,26)
        row.BackgroundColor3=Color3.fromRGB(28,18,42); row.BorderSizePixel=0
        Instance.new("UICorner", row).CornerRadius = UDim.new(0,6)
        local txt = Instance.new("TextLabel", row)
        txt.Size=UDim2.new(1,-70,1,0); txt.Position=UDim2.new(0,8,0,0)
        txt.BackgroundTransparency=1; txt.TextColor3=A2
        txt.Font=Enum.Font.GothamMedium; txt.TextSize=11
        txt.TextXAlignment=Enum.TextXAlignment.Left
        local function upd()
            txt.Text = label..": "..tostring(getter())..(suffix or "")
        end
        upd()
        local mn = Instance.new("TextButton", row)
        mn.Size=UDim2.new(0,28,1,-6); mn.Position=UDim2.new(1,-64,0,3)
        mn.BackgroundColor3=Color3.fromRGB(45,32,70); mn.Text="-"
        mn.TextColor3=Color3.new(1,1,1); mn.Font=Enum.Font.GothamBold
        mn.TextSize=12; mn.BorderSizePixel=0
        Instance.new("UICorner", mn).CornerRadius = UDim.new(0,4)
        local pl2 = Instance.new("TextButton", row)
        pl2.Size=UDim2.new(0,28,1,-6); pl2.Position=UDim2.new(1,-32,0,3)
        pl2.BackgroundColor3=Color3.fromRGB(45,32,70); pl2.Text="+"
        pl2.TextColor3=Color3.new(1,1,1); pl2.Font=Enum.Font.GothamBold
        pl2.TextSize=12; pl2.BorderSizePixel=0
        Instance.new("UICorner", pl2).CornerRadius = UDim.new(0,4)
        mn.MouseButton1Click:Connect(function()
            setter(math.max(minv, getter()-step)); upd()
            if onChange then pcall(onChange) end
        end)
        pl2.MouseButton1Click:Connect(function()
            setter(math.min(maxv, getter()+step)); upd()
            if onChange then pcall(onChange) end
        end)
        return row
    end

    local tC = aTab(T("Combat"), 55)
    local tM = aTab(T("Move"), 45)
    local tR = aTab(T("Troll"), 45)
    local tV = aTab(T("Visuals"), 60)
    local tModel = aTab(T("Model"), 55)
    local tJ = aTab(T("Jump"), 45)
    local tW = aTab(T("World"), 45)
    local tE = aTab(T("ESP"), 38)
    local tT = aTab(T("Theme"), 45)
    local tX = aTab(T("Extra"), 45)
    local function killAll()
        local c = LP.Character; if not c then notif("no character", false) return end
        local knife = c:FindFirstChild("Knife")
        if not knife then
            local bp = LP:FindFirstChildOfClass("Backpack")
            knife = bp and bp:FindFirstChild("Knife")
            if knife then
                local h = c:FindFirstChildOfClass("Humanoid")
                if h then pcall(function() h:EquipTool(knife) end) end
                task.wait(0.3)
                knife = c:FindFirstChild("Knife")
            end
        end
        if not knife then notif("need knife (murderer only)", false) return end
        local events = knife:FindFirstChild("Events")
        local stabbed = events and events:FindFirstChild("KnifeStabbed")
        local touched = events and events:FindFirstChild("HandleTouched")
        if not stabbed or not touched then notif("no knife events", false) return end
        local myp = c:FindFirstChild("HumanoidRootPart"); if not myp then return end
        local orig = myp.CFrame
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LP and alive(plr) then
                local tchar = plr.Character
                local tpart = tchar and (tchar:FindFirstChild("HumanoidRootPart") or tchar:FindFirstChild("Head"))
                if tpart then
                    pcall(function() myp.CFrame = tpart.CFrame + Vector3.new(0,0,2) end)
                    task.wait(0.1)
                    pcall(function() stabbed:FireServer() end)
                    pcall(function() touched:FireServer(tpart) end)
                    task.wait(0.05)
                end
            end
        end
        pcall(function() myp.CFrame = orig end)
    end
    if IS_MOBILE then
        local tB = aTab(T("Btns"), 55)
        local floatP = Instance.new("Frame", sg)
        floatP.Size=UDim2.new(0,124,0,0); floatP.Position=UDim2.new(1,-136,0.5,-80)
        floatP.BackgroundColor3=BG; floatP.BackgroundTransparency=0.1; floatP.BorderSizePixel=0
        floatP.Active=true; floatP.Draggable=true; floatP.Visible=false
        floatP.AutomaticSize=Enum.AutomaticSize.Y
        Instance.new("UICorner", floatP).CornerRadius = UDim.new(0,12)
        local fst = Instance.new("UIStroke", floatP); fst.Color=A; fst.Thickness=1.5; fst.Transparency=0.3
        local fll = Instance.new("UIListLayout", floatP); fll.Padding=UDim.new(0,4)
        fll.HorizontalAlignment=Enum.HorizontalAlignment.Center
        fll.SortOrder=Enum.SortOrder.LayoutOrder
        local fpd = Instance.new("UIPadding", floatP)
        fpd.PaddingTop=UDim.new(0,6); fpd.PaddingBottom=UDim.new(0,6)
        local added, floatN = {}, 0
        local function floatBtn(name, getter, setter)
            local fb = Instance.new("TextButton", floatP)
            fb.Size=UDim2.new(1,-12,0,44)
            fb.Font=Enum.Font.GothamBold; fb.TextSize=13
            fb.TextColor3=Color3.new(1,1,1)
            fb.BorderSizePixel=0; fb.AutoButtonColor=false
            fb.Text=name; fb.TextTruncate=Enum.TextTruncate.AtEnd
            Instance.new("UICorner", fb).CornerRadius = UDim.new(0,10)
            pressFx(fb)
            local function fupd()
                fb.BackgroundColor3 = getter() and Color3.fromRGB(40,160,70) or Color3.fromRGB(80,25,25)
            end
            fupd()
            fb.MouseButton1Click:Connect(function() setter(not getter()) fupd() end)
            return fb
        end
        local function floatAct(name, cb)
            local fb = Instance.new("TextButton", floatP)
            fb.Size=UDim2.new(1,-12,0,44)
            fb.BackgroundColor3=BG3; fb.Font=Enum.Font.GothamBold; fb.TextSize=13
            fb.TextColor3=A2; fb.Text=name; fb.TextTruncate=Enum.TextTruncate.AtEnd
            fb.BorderSizePixel=0; fb.AutoButtonColor=false
            Instance.new("UICorner", fb).CornerRadius = UDim.new(0,10)
            pressFx(fb)
            fb.MouseButton1Click:Connect(function() pcall(cb) end)
            return fb
        end
        local function addRow(id, name, getter, setter, isAction, cb)
            name = T(name)
            local row = Instance.new("Frame", tB)
            row.Size=UDim2.new(1,-4,0,44); row.BackgroundColor3=BG3; row.BorderSizePixel=0
            Instance.new("UICorner", row).CornerRadius = UDim.new(0,10)
            local lbl = Instance.new("TextLabel", row)
            lbl.Size=UDim2.new(1,-72,1,0); lbl.Position=UDim2.new(0,12,0,0)
            lbl.BackgroundTransparency=1; lbl.TextColor3=Color3.new(1,1,1)
            lbl.Font=Enum.Font.GothamMedium; lbl.TextSize=14
            lbl.TextXAlignment=Enum.TextXAlignment.Left
            lbl.TextTruncate=Enum.TextTruncate.AtEnd; lbl.Text=name
            local ab = Instance.new("TextButton", row)
            ab.Size=UDim2.new(0,56,0,32); ab.Position=UDim2.new(1,-62,0.5,-16)
            ab.BackgroundColor3=A; ab.Font=Enum.Font.GothamBlack; ab.TextSize=13
            ab.TextColor3=Color3.new(1,1,1); ab.Text="ADD"
            ab.BorderSizePixel=0; ab.AutoButtonColor=false
            Instance.new("UICorner", ab).CornerRadius = UDim.new(0,8)
            pressFx(ab)
            ab.MouseButton1Click:Connect(function()
                if added[id] then
                    pcall(function() added[id]:Destroy() end)
                    added[id]=nil; floatN=floatN-1; ab.Text="ADD"
                else
                    if isAction then added[id]=floatAct(name, cb)
                    else added[id]=floatBtn(name, getter, setter) end
                    floatN=floatN+1; ab.Text="DEL"
                end
                floatP.Visible = floatN > 0
            end)
        end
        local function togRow(id, name, getter, setter)
            addRow(id, name, getter, setter, false, nil)
        end
        local function actRow(id, name, cb)
            addRow(id, name, nil, nil, true, cb)
        end
        togRow("sa","Silent Aim", function() return SA.enabled end, function(v) SA.enabled=v end)
        togRow("ka","Kill Aura", function() return KA.on end, function(v) KA.on=v end)
        togRow("ag","Auto Gun", function() return autograb_on end, function(v)
            autograb_on=v
            if v then if not grab_desc_conn then grab_desc_conn=workspace.DescendantAdded:Connect(grab_desc_added) end task.spawn(scan_guns)
            else grab_desc_stop() end
        end)
        togRow("fly","Fly", function() return MOVE.fly end, function(v) MOVE.fly=v if v then startFly() else stopFly() end end)
        togRow("nc","Noclip", function() return MOVE.noclip end, function(v) MOVE.noclip=v if v then startNoclip() else stopNoclip() end end)
        togRow("esp","ESP", function() return fl.esp end, function(v) fl.esp=v if v then mkESP() else clESP() end end)
        togRow("spin","Spin", function() return MOVE.spin end, function(v) MOVE.spin=v if v then startSpin() else stopSpin() end end)
        togRow("ij","Infinite Jump", function() return MOVE.infJump end, function(v) MOVE.infJump=v if v then startInfJump() else stopInfJump() end end)
        actRow("fm","Fling Murderer", function()
            for _, p in ipairs(Players:GetPlayers()) do
                if p ~= LP and alive(p) and cachedRole(p) == "m" then flingTarget(p) return end
            end
        end)
        actRow("fs","Fling Sheriff", function()
            for _, p in ipairs(Players:GetPlayers()) do
                if p ~= LP and alive(p) and cachedRole(p) == "s" then flingTarget(p) return end
            end
        end)
        actRow("kall","Kill All", function() task.spawn(killAll) end)
        local function flyHold(txt, pos, key)
            local hb = Instance.new("TextButton", sg)
            hb.Size=UDim2.new(0,60,0,60); hb.Position=pos
            hb.BackgroundColor3=BG3; hb.TextColor3=A2
            hb.Font=Enum.Font.GothamBlack; hb.TextSize=24
            hb.Text=txt; hb.BorderSizePixel=0; hb.AutoButtonColor=false
            hb.Visible=false
            Instance.new("UICorner", hb).CornerRadius = UDim.new(1,0)
            local hs = Instance.new("UIStroke", hb); hs.Color=A; hs.Thickness=2; hs.Transparency=0.3
            hb.MouseButton1Down:Connect(function() MOVE[key]=true end)
            hb.MouseButton1Up:Connect(function() MOVE[key]=false end)
            hb.MouseLeave:Connect(function() MOVE[key]=false end)
            return hb
        end
        MOVE.upBtn = flyHold("в–І", UDim2.new(1,-76,0.5,60), "flyUp")
        MOVE.dnBtn = flyHold("в–ј", UDim2.new(1,-76,0.5,130), "flyDown")
    end

    mkToggle(tC,"Silent Aim",     function() return SA.enabled end, function(v) SA.enabled=v end)
    mkToggle(tC,"Silent Predict", function() return SA.predict end, function(v) SA.predict=v end)
    mkToggle(tC,"WallCheck", function() return SA.walls end, function(v) SA.walls=v end)
    mkToggle(tC,"Kill Aura",      function() return KA.on end,      function(v) KA.on=v end)
    mkBtn(tC,"Kill All", function() task.spawn(killAll) end)
    mkToggle(tC,"grab gun", function() return autograb_on end, function(v)
        autograb_on = v
        if v then
            if not grab_desc_conn then
                grab_desc_conn = workspace.DescendantAdded:Connect(grab_desc_added)
            end
            task.spawn(scan_guns)
        else
            grab_desc_stop()
        end
    end)
    mkToggle(tC,"Bullet Tracer",  function() return TRACE.on end,   function(v)
        TRACE.on = v
        if v then task.spawn(connect_trace_fn) end
    end, connect_trace_fn, connect_trace_fn, "tracer")
    mkBtn(tC,"Tracer Settings", function() showSet("tracer", function() end) end)

    mkToggle(tM,"Walkspeed", function() return MOVE.speedOn end, function(v) MOVE.speedOn=v; applySpeed() end, nil, nil, nil, nil,
        {t="Speed", get=function() return MOVE.speed end, set=function(v) MOVE.speed=v end, step=2, min=1, max=500, apply=applySpeed})
    mkNum(tM,"Speed", function() return MOVE.speed end, function(v) MOVE.speed=v end, 2, 1, 500, "", applySpeed)
    mkToggle(tM,"JumpPower", function() return MOVE.jumpOn end, function(v) MOVE.jumpOn=v; applyJump() end, nil, nil, nil, nil,
        {t="Power", get=function() return MOVE.jump end, set=function(v) MOVE.jump=v end, step=5, min=1, max=500, apply=applyJump})
    mkNum(tM,"Power", function() return MOVE.jump end, function(v) MOVE.jump=v end, 5, 1, 500, "", applyJump)
    mkToggle(tM,"Infinite Jump", function() return MOVE.infJump end, function(v)
        MOVE.infJump = v
        if v then startInfJump() else stopInfJump() end
    end)
    mkToggle(tM,"HipHeight", function() return MOVE.hipOn end, function(v) MOVE.hipOn=v; applyHip() end, nil, nil, nil, nil,
        {t="Hip", get=function() return MOVE.hip end, set=function(v) MOVE.hip=v end, step=0.5, min=-5, max=50, apply=applyHip})
    mkNum(tM,"Hip", function() return MOVE.hip end, function(v) MOVE.hip=v end, 0.5, -5, 50, "", applyHip)
    mkToggle(tM,"Noclip", function() return MOVE.noclip end, function(v)
        MOVE.noclip = v
        if v then startNoclip() else stopNoclip() end
    end)
    mkToggle(tM,"Fly", function() return MOVE.fly end, function(v)
        MOVE.fly = v
        if v then startFly() else stopFly() end
    end, nil, nil, nil, nil,
        {t="FlySpeed", get=function() return MOVE.flySpeed end, set=function(v) MOVE.flySpeed=v end, step=5, min=5, max=500})
    mkNum(tM,"FlySpeed", function() return MOVE.flySpeed end, function(v) MOVE.flySpeed=v end, 5, 5, 500, "")
    mkToggle(tM,"Spin", function() return MOVE.spin end, function(v)
        MOVE.spin = v
        if v then startSpin() else stopSpin() end
    end, nil, nil, nil, nil,
        {t="SpinSpeed", get=function() return MOVE.spinSpeed end, set=function(v) MOVE.spinSpeed=v end, step=90, min=30, max=3000})
    mkNum(tM,"SpinSpeed", function() return MOVE.spinSpeed end, function(v) MOVE.spinSpeed=v end, 90, 30, 3000, "/s")
    mkToggle(tM,"Custom Gravity", function() return MOVE.gravOn end, function(v) MOVE.gravOn=v; applyGrav() end, nil, nil, nil, nil,
        {t="Gravity", get=function() return math.floor(MOVE.grav) end, set=function(v) MOVE.grav=v end, step=20, min=0, max=1000, apply=applyGrav})
    mkNum(tM,"Gravity", function() return math.floor(MOVE.grav) end, function(v) MOVE.grav=v end, 20, 0, 1000, "", applyGrav)
    mkBtn(tM,"Reset Move", moveReset)
    mkBtn(tM,"TP to Map", function()
        local best, bd = nil, math.huge
        local hrp = HRP() if not hrp then return end
        for _,o in ipairs(workspace:GetDescendants()) do
            if o:IsA("SpawnLocation") and o.Parent and not string.find(tostring(o:GetFullName()), "Lobby") then
                local d = (o.Position - hrp.Position).Magnitude
                if d < bd then bd = d best = o end
            end
        end
        if best then hrp.CFrame = best.CFrame + Vector3.new(0,5,0) end
    end)
    mkBtn(tM,"TP to Lobby", function()
        local hrp = HRP() if not hrp then return end
        for _,o in ipairs(workspace:GetDescendants()) do
            if o:IsA("SpawnLocation") and string.find(tostring(o:GetFullName()), "Lobby") then
                hrp.CFrame = o.CFrame + Vector3.new(0,5,0) break
            end
        end
    end)
    mkToggle(tM,"Anti Fall (void)", function() return MOVE.antiFall==true end, function(v) MOVE.antiFall=v end)
    mkToggle(tM,"Anti Fling", function() return fl.antifling==true end, function(v) fl.antifling=v end)
    mkToggle(tM,"Ctrl+Click TP", function() return MOVE.ctrltp==true end, function(v) MOVE.ctrltp=v end)

    mkBtn(tR,"Fling Nearest", flingNearest)
    mkBtn(tR,"Fling All",     flingAll)
    mkBtn(tR,"Fling Murderer", function()
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and alive(p) and cachedRole(p) == "m" then flingTarget(p) return end
        end
        notif("no murderer", false)
    end)
    mkBtn(tR,"Fling Sheriff", function()
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and alive(p) and cachedRole(p) == "s" then flingTarget(p) return end
        end
        notif("no sheriff", false)
    end)
    mkBtn(tR,"Stop All Troll", stopTroll)
    mkBtn(tR,"TP to Murder", function()
        for _,p in ipairs(Players:GetPlayers()) do
            if p~=LP and alive(p) and cachedRole(p)=="m" then
                local hrp=HRP() local th=p.Character and p.Character:FindFirstChild("HumanoidRootPart")
                if hrp and th then hrp.CFrame=th.CFrame+Vector3.new(0,0,3) end
                break
            end
        end
    end)
    mkBtn(tR,"TP to Sheriff", function()
        for _,p in ipairs(Players:GetPlayers()) do
            if p~=LP and alive(p) and cachedRole(p)=="s" then
                local hrp=HRP() local th=p.Character and p.Character:FindFirstChild("HumanoidRootPart")
                if hrp and th then hrp.CFrame=th.CFrame+Vector3.new(0,0,3) end
                break
            end
        end
    end)
    mkBtn(tR,"TP to Random", function()
        local all = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and alive(p) then all[#all+1] = p end
        end
        if #all == 0 then return end
        local tgt = all[math.random(1, #all)]
        local hrp = HRP()
        local th = tgt.Character and tgt.Character:FindFirstChild("HumanoidRootPart")
        if hrp and th then hrp.CFrame = th.CFrame + Vector3.new(0,0,3) end
    end)
    mkBtn(tR,"Dizzy (spin+jump)", function()
        local h=hum() if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
        MOVE.spin=true pcall(startSpin)
        task.delay(3,function() MOVE.spin=false pcall(stopSpin) end)
    end)

    mkToggle(tV,"Neon Trail", function() return fl.trail end, function(v) fl.trail=v; if v then mkTrail() else clTrail() end end, mkTrail, clTrail, "trail")
    mkToggle(tV,"China Hat",  function() return fl.hat end,   function(v) fl.hat=v;   if v then mkHat()  else clHat()  end end, mkHat, clHat, "hat")
    mkToggle(tV,"Body Glow",  function() return fl.glow end,  function(v) fl.glow=v;  if v then mkGlow() else clGlow() end end, mkGlow, clGlow, "glow")
    mkToggle(tV,"Rainbow",    function() return fl.rainbow end, function(v) fl.rainbow=v; if v then mkRB() else clRB() end end, mkRB, clRB, "rainbow")

    mkToggle(tV,"Aura Particles", function() return fl.parts end, function(v) fl.parts=v; if v then mkParts() else clParts() end end, mkParts, clParts, "parts")
    mkToggle(tV,"Sky Dust", function() return fl.skyDust end, function(v) fl.skyDust=v; if v then mkSky() else clSky() end end, mkSky, clSky, "skyDust")
    mkToggle(tV,"Snow Aura", function() return fl.snow end, function(v) fl.snow=v; if v then mkSnow() else clSnow() end end, mkSnow, clSnow, "snow")
    mkToggle(tV,"Bounce Particles", function() return fl.bounce end, function(v) fl.bounce=v; if v then mkBo() else clBo() end end, mkBo, clBo, "bounce")
    mkToggle(tV,"Lightning Aura", function() return fl.lightning end, function(v) fl.lightning=v; if v then mkLight() else clLight() end end, mkLight, clLight, "lightning")
    mkToggle(tV,"Orb Aura", function() return fl.orbAura end, function(v) fl.orbAura=v; if v then mkOrb() else clOrb() end end, mkOrb, clOrb, "orbAura")
    mkToggle(tV,"Ghost Trail", function() return fl.ghost end, function(v) fl.ghost=v; if v then mkGhost() else clGhost() end end, mkGhost, clGhost, "ghost")
    mkToggle(tV,"Footsteps", function() return fl.foot end, function(v) fl.foot=v; if v then mkFoot() else clFoot() end end, mkFoot, clFoot, "foot")
    mkToggle(tModel,"Headless", function() return fl.headless==true end, function(v) fl.headless=v; pcall(applySkinAll) end)

    mkToggle(tJ,"Jump Circles", function() return fl.jump end, function(v) fl.jump=v; if v then mkJump() else clJump() end end, mkJump, clJump, "jump")
    mkBtn(tJ,"Classic",        function() jStyle = "Classic" end)
    mkBtn(tJ,"Double Ring",    function() jStyle = "Double" end)
    mkBtn(tJ,"Triple Ripple",  function() jStyle = "Ripple" end)
    mkBtn(tJ,"Particle Burst", function() jStyle = "Particles" end)

    mkToggle(tW,"Cyberpunk Atmo", function() return fl.atmo end, function(v) fl.atmo=v; if v then mkAtmo() else clAtmo() end end, mkAtmo, clAtmo, "atmo")
    mkToggle(tW,"FullBright", function() return fl.fb end, function(v) fl.fb=v; if v then mkFB() else clFB() end end)
    mkToggle(tW,"Skybox", function() return fl.sky end, function(v) fl.sky=v; if v then mkSkybox() else clSkybox() end end)
    mkToggle(tW,"Bloom", function() return fl.bloom end, function(v) fl.bloom=v; if v then mkBloom() else clBloom() end end, mkBloom, clBloom, "bloom")
    mkBtn(tW,"Day",    function() applyTime("Day") end)
    mkBtn(tW,"Sunset", function() applyTime("Sunset") end)
    mkBtn(tW,"Night",  function() applyTime("Night") end)
    mkBtn(tW,"Cyber",  function() applyTime("Cyber") end)
    mkToggle(tW,"Fog", function() return fl._fog end, function(v) fl._fog=v; toggleFog(v) end)

    mkToggle(tE,"Role ESP", function() return fl.esp end, function(v) fl.esp=v; if v then mkESP() else clESP() end end, mkESP, clESP, "esp")
    mkToggle(tE,"Ghost ESP", function() return fl.espGhost end, function(v) fl.espGhost = v end)

    mkToggle(tX,"Kill Lightning (Murderer)", function() return killOn end, function(v) killOn=v end)
    mkToggle(tX,"Backtrack", function() return BT.on end, function(v)
        BT.on = v
        if v then task.spawn(bt_build) end
    end)
    mkToggle(tX,"Anti AFK", function() return fl.noafk==true end, function(v) fl.noafk=v end)
    mkBtn(tX,"Reset colors", function()
        for k,_ in pairs(FC) do FC[k] = A end
    end)
    mkBtn(tX,"Clear particles", function()
        for _, p in ipairs(workspace:GetDescendants()) do
            if p:IsA("ParticleEmitter") then p:Destroy() end
        end
    end)

    for name, _ in pairs(TH) do
        mkBtn(tT, name, function()
            cur = name
            local Th = TH[cur]
            A,A2,BG,BG2,BG3,ON,OFF = Th.A,Th.A2,Th.BG,Th.BG2,Th.BG3,Th.ON,Th.OFF
            main.BackgroundColor3 = BG
            mst.Color = A
            tt.BackgroundColor3 = BG2
            tt.TextColor3 = A
            sb.BackgroundColor3 = BG2
            veil.BackgroundColor3 = A
            accent.BackgroundColor3 = A
            logo.BackgroundColor3 = A
            ttg.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, BG2), ColorSequenceKeypoint.new(1, BG)})
            mg.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, BG), ColorSequenceKeypoint.new(1, Color3.fromRGB(4,7,14))})
            openBtn.BackgroundColor3 = A
            obStroke.Color = A2
        end)
    end
    mkBtn(tT,"Save", function()
        _G.UN_ADDON_CFG = {
            fl=fl, FC=FC, SZ=SZ, jStyle=jStyle, cur=cur,
            skyHeart=skyHeart, boHeart=boHeart, timePreset=timePreset,
            move = {
                speedOn=MOVE.speedOn, speed=MOVE.speed,
                jumpOn=MOVE.jumpOn,   jump=MOVE.jump,
                hipOn=MOVE.hipOn,     hip=MOVE.hip,
                infJump=MOVE.infJump, noclip=MOVE.noclip,
                fly=MOVE.fly,         flySpeed=MOVE.flySpeed,
                gravOn=MOVE.gravOn,   grav=MOVE.grav,
                spin=MOVE.spin,       spinSpeed=MOVE.spinSpeed,
            },
            bt = { on = BT.on },
        }
        print("[addon] saved")
    end)
    mkBtn(tT,"Load", function()
        if _G.UN_ADDON_CFG then
            for k,v in pairs(_G.UN_ADDON_CFG.fl) do fl[k]=v end
            for k,v in pairs(_G.UN_ADDON_CFG.FC) do FC[k]=v end
            for k,v in pairs(_G.UN_ADDON_CFG.SZ) do SZ[k]=v end
            jStyle = _G.UN_ADDON_CFG.jStyle or "Classic"
            cur = _G.UN_ADDON_CFG.cur or cur
            skyHeart = _G.UN_ADDON_CFG.skyHeart or false
            boHeart = _G.UN_ADDON_CFG.boHeart or false
            timePreset = _G.UN_ADDON_CFG.timePreset or "Day"
            applyTime(timePreset)
            if _G.UN_ADDON_CFG.move then
                for k,v in pairs(_G.UN_ADDON_CFG.move) do MOVE[k]=v end
                if MOVE.infJump then startInfJump() end
                if MOVE.noclip then startNoclip() end
                if MOVE.fly then startFly() end
                if MOVE.spin then startSpin() end
                applySpeed(); applyJump(); applyHip(); applyGrav()
            end
            if _G.UN_ADDON_CFG.bt then
                BT.on = _G.UN_ADDON_CFG.bt.on or false
                if BT.on then task.spawn(bt_build) end
            end
        end
    end)

    task.defer(function()
        loadBinds()
    end)

    print("[starware] GUI ready")
end)

-- ============================================================
-- BIND LISTENER
-- ============================================================
ADDON.bindConn = UIS.InputBegan:Connect(function(input, gp)
    if bindPicking then
        if input.KeyCode == Enum.KeyCode.Escape then
            notif("bind cancelled", false)
            bindPicking = nil
            return
        end
        if string.find(tostring(input.UserInputType), "MouseButton") and BINDS._ignoreMouseUntil and os.clock() < BINDS._ignoreMouseUntil then
            return
        end
        local key = bindPicking.key
        local entry = BINDS[key]
        if entry then
            entry.input = input.KeyCode ~= Enum.KeyCode.Unknown and input.KeyCode or input.UserInputType
            saveBinds()
            notif(entry.name.." -> "..inputName(entry.input), true)
            if entry.onUpdate then pcall(entry.onUpdate) end
        end
        bindPicking = nil
        return
    end
    if input.UserInputType == Enum.UserInputType.MouseButton1 and MOVE.ctrltp and not getgenv().UN_MOBILE then
        if UIS:IsKeyDown(Enum.KeyCode.LeftControl) or UIS:IsKeyDown(Enum.KeyCode.RightControl) then
            local hrp = HRP()
            local m = LP:GetMouse()
            if hrp and m and m.Hit then
                pcall(function() hrp.CFrame = CFrame.new(m.Hit.Position + Vector3.new(0,5,0)) end)
            end
            return
        end
    end
    if gp then return end
    for k, v in pairs(BINDS) do
        if type(v)=="table" and v.input then
            local match = false
            if input.UserInputType == v.input then match = true end
            if typeof(v.input) == "EnumItem" and v.input.EnumType == Enum.KeyCode and input.KeyCode == v.input and #v.input.Name > 1 then match = true end
            if match and v.getter and v.setter then
                local okg, cur = pcall(v.getter)
                if okg then
                    pcall(v.setter, not cur)
                    if v.onUpdate then pcall(v.onUpdate) end
                    pcall(notif, v.name, not cur)
                end
            end
        end
    end
end)

local mouse = LP:GetMouse()
ADDON.keyConn = mouse.KeyDown:Connect(function(keyName)
    if bindPicking then return end
    if UIS:GetFocusedTextBox() ~= nil then return end
    local kn = string.upper(tostring(keyName or ""))
    if #kn ~= 1 then return end
    local digitKC = { ["0"]="Zero", ["1"]="One", ["2"]="Two", ["3"]="Three", ["4"]="Four", ["5"]="Five", ["6"]="Six", ["7"]="Seven", ["8"]="Eight", ["9"]="Nine" }
    local kc = Enum.KeyCode[digitKC[kn] or kn]
    if kc == nil then return end
    for k, v in pairs(BINDS) do
        if type(v)=="table" and v.input and v.input == kc then
            if v.getter and v.setter then
                local okg, cur = pcall(v.getter)
                if okg then
                    pcall(v.setter, not cur)
                    if v.onUpdate then pcall(v.onUpdate) end
                    pcall(notif, v.name, not cur)
                end
            end
        end
    end
end)

ADDON.idleConn = LP.Idled:Connect(function()
    if fl.noafk and ADDON.loaded then
        pcall(function()
            local vu = game:GetService("VirtualUser")
            vu:CaptureController()
            vu:ClickButton2(Vector2.new())
        end)
    end
end)

-- ============================================================
-- MAIN LOOP
-- ============================================================
ADDON.hb = RS.Heartbeat:Connect(function()
    if not ADDON.loaded then return end
    pcall(function()
        local now = os.clock()
        if now - (ADDON.role_acc or 0) >= 0.2 then
            ADDON.role_acc = now
            refresh_target()
        end
        sample_ping()
        track_step(now)
        ka_tick()
        troll_tick()
        if fl.antifling and not MOVE.fly then
            local ac = LP.Character
            local ahrp = ac and ac:FindFirstChild("HumanoidRootPart")
            if ahrp then
                pcall(function()
                    if ahrp.AssemblyLinearVelocity.Magnitude > 60 then
                        ahrp.AssemblyLinearVelocity = Vector3.zero
                        ahrp.Velocity = Vector3.zero
                    end
                end)
                pcall(function()
                    ahrp.AssemblyAngularVelocity = Vector3.zero
                    ahrp.RotVelocity = Vector3.zero
                end)
            end
        end
        if now - (ADDON.hook_acc or 0) >= 1 then
            ADDON.hook_acc = now
            install_hooks()
            connect_trace_fn()
        end
    end)
end)

task.spawn(function()
    while ADDON.loaded do
        pcall(hookKills)
        task.wait(0.5)
    end
end)

ADDON.charConn = LP.CharacterAdded:Connect(function()
    task.wait(0.5)
    if not ADDON.loaded then return end
    applySpeed(); applyJump(); applyHip()
    if MOVE.fly then pcall(stopFly) pcall(startFly) end
    if MOVE.noclip then pcall(stopNoclip) pcall(startNoclip) end
    if MOVE.spin then pcall(startSpin) end
    if MOVE.infJump then pcall(startInfJump) end
    if fl.trail then pcall(mkTrail) end
    if fl.parts then pcall(mkParts) end
    if fl.skyDust then pcall(mkSky) end
    if fl.snow then pcall(mkSnow) end
    if fl.bounce then pcall(mkBo) end
    if fl.hat then pcall(mkHat) end
    if fl.lightning then pcall(mkLight) end
    if fl.orbAura then pcall(mkOrb) end
    if fl.ghost then pcall(mkGhost) end
    if fl.foot then pcall(mkFoot) end
    if fl.rainbow then pcall(mkRB) end
    if fl.glow then pcall(mkGlow) end
    pcall(applySkinAll)
    if fl.tracer then pcall(enableTracer_fn) end
    if fl.jump then pcall(mkJump) end
    if BT.on then pcall(bt_build) end
end)

task.spawn(function()
    pcall(install_hooks)
    pcall(connect_trace_fn)
    if fl.tracer then pcall(enableTracer_fn) end
    if MOVE.infJump then startInfJump() end
    if MOVE.noclip then startNoclip() end
    if MOVE.fly then startFly() end
    if MOVE.spin then startSpin() end
end)

-- ============================================================
-- UNLOAD
-- ============================================================
ADDON.unload = function()
    ADDON.loaded = false
    SA.enabled=false; KS.enabled=false; KA.on=false
    TRACE.on=false
    autograb_on = false
    BT.on = false
    grab_desc_stop()
    getgenv().KNIFE_AIM_RESOLVE = nil

    moveReset()
    stopTroll()
    workspace.Gravity = 196.2

    if ADDON.hb then pcall(function() ADDON.hb:Disconnect() end); ADDON.hb=nil end
    if ADDON.bindConn then pcall(function() ADDON.bindConn:Disconnect() end); ADDON.bindConn=nil end
    if ADDON.keyConn then pcall(function() ADDON.keyConn:Disconnect() end); ADDON.keyConn=nil end
    if ADDON.idleConn then pcall(function() ADDON.idleConn:Disconnect() end); ADDON.idleConn=nil end
    if TRACE.conn then pcall(function() TRACE.conn:Disconnect() end); TRACE.conn=nil end
    if ADDON.charConn then pcall(function() ADDON.charConn:Disconnect() end); ADDON.charConn=nil end
    if bt_conn then pcall(function() bt_conn:Disconnect() end); bt_conn=nil end
    pcall(bt_destroy)

    for _, fn in ipairs({clTrail, clParts, clSky, clSnow, clBo, clHat, clLight,
                         clOrb, clGhost, clFoot, clRB, clGlow, clJump, clAtmo,
                         clFB, clSkybox, clBloom, clESP, disableTracer_fn}) do
        pcall(fn)
    end
    pcall(function() if fl._fog then toggleFog(false) end end)

    local m = WSH.m
    if m then
        pcall(function() setreadonly(m,false) end)
        if WSH.om then pcall(function() m.GetMouseTargetCFrame = WSH.om end) end
        if WSH.os then pcall(function() m.GetTargetPosition = WSH.os end) end
    end

    local PlayerGui = LP:FindFirstChild("PlayerGui")
    if PlayerGui then
        local old = PlayerGui:FindFirstChild("UN_ADDON_GUI")
        if old then pcall(function() old:Destroy() end) end
        local oldT = PlayerGui:FindFirstChild("UN_ADDON_TOASTS")
        if oldT then pcall(function() oldT:Destroy() end) end
    end
    getgenv().UN_ADDON = nil
    print("[addon] unloaded")
end

print("[addon] ready")