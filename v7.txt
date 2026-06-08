-- 原神V2 --此脚本不加密，所以说呢，如果要二改发布请在脚本里提及WeiXun。谢谢你！--
local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()
local Options = Library.Options
local Toggles = Library.Toggles
Library.ForceCheckbox = false
Library.ShowToggleFrameInKeybinds = true

local winTitle = string.char(229, 142, 159, 231, 165, 158, 86, 50) -- 原神V2
local winFooter = string.char(77, 97, 100, 101, 32, 98, 121, 32, 87, 101, 105, 88, 117, 110) -- Made by WeiXun

local Window = Library:CreateWindow({
    Title = winTitle,
    Footer = winFooter,
    NotifySide = "Right",
    ShowCustomCursor = true,
})

local Tabs = {
    Combat = Window:AddTab("战斗", "crosshair"),
    SilentAim = Window:AddTab("静默自瞄", "target"),
    Skins = Window:AddTab("皮肤", "swords"),
    Visuals = Window:AddTab("视觉", "eye"),
    ["UI Settings"] = Window:AddTab("界面设置", "settings"),
}

-- Services
local RS = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local CAS = game:GetService("ContextActionService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TextService = game:GetService("TextService")
local GuiService = game:GetService("GuiService")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local CharactersFolder = workspace:WaitForChild("Characters", 10)
local Mouse = player:GetMouse()

-- Shared functions
local getTFolder = function() return CharactersFolder:FindFirstChild("Terrorists") end
local getCTFolder = function() return CharactersFolder:FindFirstChild("Counter-Terrorists") end
local isAlive = function()
    local t, ct = getTFolder(), getCTFolder()
    return (t and t:FindFirstChild(player.Name)) or (ct and ct:FindFirstChild(player.Name))
end
local getEnemyFolder = function()
    if not isAlive() then return nil end
    local t, ct = getTFolder(), getCTFolder()
    if t and t:FindFirstChild(player.Name) then return ct end
    if ct and ct:FindFirstChild(player.Name) then return t end
    return nil
end
local wallCheckParams = RaycastParams.new()
wallCheckParams.FilterType = Enum.RaycastFilterType.Exclude
local IsVisible = function(p)
    if not p then return false end
    local c = player.Character
    if not c then return false end
    local h = c:FindFirstChild("Head")
    if not h then return false end
    local el = {c}
    if camera then table.insert(el, camera) end
    wallCheckParams.FilterDescendantsInstances = el
    local r = Workspace:Raycast(h.Position, (p.Position - h.Position).Unit * 1000, wallCheckParams)
    if r then return r.Instance:IsDescendantOf(p.Parent) end
    return true
end

local hotkeyValues = {"LeftAlt","LeftShift","RightShift","LeftControl","RightControl","Q","E","R","F","X","C","V","鼠标右键","鼠标左键","鼠标中键"}
local hotkeyMap = {
    ["LeftAlt"]=Enum.KeyCode.LeftAlt, ["LeftShift"]=Enum.KeyCode.LeftShift,
    ["RightShift"]=Enum.KeyCode.RightShift, ["LeftControl"]=Enum.KeyCode.LeftControl,
    ["RightControl"]=Enum.KeyCode.RightControl, ["Q"]=Enum.KeyCode.Q, ["E"]=Enum.KeyCode.E,
    ["R"]=Enum.KeyCode.R, ["F"]=Enum.KeyCode.F, ["X"]=Enum.KeyCode.X, ["C"]=Enum.KeyCode.C,
    ["V"]=Enum.KeyCode.V, ["鼠标右键"]=Enum.UserInputType.MouseButton2,
    ["鼠标左键"]=Enum.UserInputType.MouseButton1, ["鼠标中键"]=Enum.UserInputType.MouseButton3,
}
local getHotkeyName = function(k) for n,v in pairs(hotkeyMap) do if v==k then return n end end return "鼠标右键" end

-- ==================== COMBAT ====================
local VisualAimbot = {Enabled = false, ShowFOV = false, Radius = 100, Smooth = 3, Mode = "自动", WallCheck = false, Key = Enum.UserInputType.MouseButton2, KeyHeld = false}
local FOVCircle = Drawing.new("Circle")
FOVCircle.Position = Vector2.new(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
FOVCircle.Radius = VisualAimbot.Radius
FOVCircle.Filled = false
FOVCircle.Color = Color3.fromRGB(255,255,255)
FOVCircle.Visible = false
FOVCircle.Thickness = 1

local getClosestEnemy = function()
    local cl, sd = nil, VisualAimbot.Radius
    local ef = getEnemyFolder() if not ef or not VisualAimbot.Enabled then return nil end
    local mp = UserInputService:GetMouseLocation()
    for _, e in ipairs(ef:GetChildren()) do
        local hum = e:FindFirstChildOfClass("Humanoid") local hd = e:FindFirstChild("Head")
        if hum and hum.Health>0 and hd then
            if VisualAimbot.WallCheck and not IsVisible(hd) then continue end
            local hp, on = camera:WorldToViewportPoint(hd.Position)
            if on then
                local d = (Vector2.new(hp.X, hp.Y) - mp).Magnitude
                if d < sd then sd = d; cl = hd end
            end
        end
    end
    return cl
end

UserInputService.InputBegan:Connect(function(i) if i.UserInputType == VisualAimbot.Key or i.KeyCode == VisualAimbot.Key then VisualAimbot.KeyHeld = true end end)
UserInputService.InputEnded:Connect(function(i) if i.UserInputType == VisualAimbot.Key or i.KeyCode == VisualAimbot.Key then VisualAimbot.KeyHeld = false end end)

RunService.RenderStepped:Connect(function()
    if VisualAimbot.ShowFOV then
        FOVCircle.Position = UserInputService:GetMouseLocation(); FOVCircle.Radius = VisualAimbot.Radius; FOVCircle.Visible = true
    else FOVCircle.Visible = false end
    if not VisualAimbot.Enabled or not isAlive() then return end
    local aim = (VisualAimbot.Mode == "自动") or (VisualAimbot.Mode == "热键" and VisualAimbot.KeyHeld)
    if not aim then return end
    local t = getClosestEnemy()
    if t and mousemoverel then
        local hp = camera:WorldToViewportPoint(t.Position) local mp = UserInputService:GetMouseLocation()
        mousemoverel((hp.X - mp.X)/VisualAimbot.Smooth, (hp.Y - mp.Y)/VisualAimbot.Smooth)
    end
end)

local CombatGroup = Tabs.Combat:AddLeftGroupbox("视觉自瞄", "target")
CombatGroup:AddToggle("AimbotToggle", { Text = "启用视觉自瞄", Default = false, Callback = function(v) VisualAimbot.Enabled = v end })
CombatGroup:AddDropdown("AimbotMode", { Text = "自瞄模式", Values = {"自动","热键"}, Default = "自动", Callback = function(v) VisualAimbot.Mode = v end })
CombatGroup:AddDropdown("AimbotHotkey", { Text = "自瞄热键", Values = hotkeyValues, Default = getHotkeyName(VisualAimbot.Key), Callback = function(v) VisualAimbot.Key = hotkeyMap[v] or Enum.UserInputType.MouseButton2 end })
CombatGroup:AddToggle("AimbotWallCheck", { Text = "墙壁检测", Default = false, Callback = function(v) VisualAimbot.WallCheck = v end })
CombatGroup:AddToggle("FOVToggle", { Text = "显示FOV圈", Default = false, Callback = function(v) VisualAimbot.ShowFOV = v end })
CombatGroup:AddSlider("FOVSlider", { Text = "FOV半径", Default = 100, Min = 10, Max = 500, Rounding = 0, Suffix = "px", Callback = function(v) VisualAimbot.Radius = v end })
CombatGroup:AddSlider("AimbotSmoothing", { Text = "平滑度", Default = 3, Min = 1, Max = 10, Rounding = 0, Suffix = "", Callback = function(v) VisualAimbot.Smooth = v end })

-- TriggerBot
local TriggerBot = {Enabled = false, Delay = 0, Mode = "自动", WallCheck = false, Key = Enum.KeyCode.E, KeyHeld = false}
UserInputService.InputBegan:Connect(function(i) if i.UserInputType == TriggerBot.Key or i.KeyCode == TriggerBot.Key then TriggerBot.KeyHeld = true end end)
UserInputService.InputEnded:Connect(function(i) if i.UserInputType == TriggerBot.Key or i.KeyCode == TriggerBot.Key then TriggerBot.KeyHeld = false end end)

local TriggerGroup = Tabs.Combat:AddLeftGroupbox("自动扳机", "target")
TriggerGroup:AddToggle("TriggerBotToggle", { Text = "启用自动扳机", Default = false, Callback = function(v) TriggerBot.Enabled = v end })
TriggerGroup:AddDropdown("TriggerBotMode", { Text = "扳机模式", Values = {"自动","热键"}, Default = "自动", Callback = function(v) TriggerBot.Mode = v end })
TriggerGroup:AddDropdown("TriggerBotHotkey", { Text = "扳机热键", Values = hotkeyValues, Default = getHotkeyName(TriggerBot.Key), Callback = function(v) TriggerBot.Key = hotkeyMap[v] or Enum.KeyCode.E end })
TriggerGroup:AddToggle("TriggerBotWallCheck", { Text = "墙壁检测", Default = false, Callback = function(v) TriggerBot.WallCheck = v end })
TriggerGroup:AddSlider("TriggerBotDelay", { Text = "射击延迟", Default = 0, Min = 0, Max = 500, Rounding = 0, Suffix = "ms", Callback = function(v) TriggerBot.Delay = v end })

task.spawn(function()
    while task.wait(0.01) do
        local shoot = false
        if TriggerBot.Enabled and isAlive() then
            if TriggerBot.Mode == "自动" then shoot = true elseif TriggerBot.Mode == "热键" then shoot = TriggerBot.KeyHeld end
        end
        if not shoot then continue end
        local ray = camera:ViewportPointToRay(camera.ViewportSize.X/2, camera.ViewportSize.Y/2)
        local params = RaycastParams.new(); params.FilterType = Enum.RaycastFilterType.Exclude
        local ignore = {camera}; if player.Character then table.insert(ignore, player.Character) end
        params.FilterDescendantsInstances = ignore
        local result = Workspace:Raycast(ray.Origin, ray.Direction*1000, params)
        if result and result.Instance then
            local model = result.Instance:FindFirstAncestorOfClass("Model")
            if model and model:FindFirstChildOfClass("Humanoid") then
                local ef = getEnemyFolder()
                if ef and model.Parent == ef then
                    local hum = model:FindFirstChildOfClass("Humanoid")
                    if hum and hum.Health > 0 then
                        if TriggerBot.WallCheck and not IsVisible(model:FindFirstChild("Head")) then continue end
                        if TriggerBot.Delay > 0 then task.wait(TriggerBot.Delay/1000) end
                        if mouse1click then mouse1click() end
                        task.wait(0.05)
                    end
                end
            end
        end
    end
end)

-- Hitbox Expander
local Hitbox = {Enabled = false, Size = 3}
local originalHeadSizes = {}
local HitboxGroup = Tabs.Combat:AddLeftGroupbox("命中框扩大", "target")
HitboxGroup:AddToggle("HitboxToggle", { Text = "启用命中框扩大", Default = false, Callback = function(v) Hitbox.Enabled = v end })
HitboxGroup:AddSlider("HitboxSize", { Text = "大小", Default = 3, Min = 1, Max = 3, Rounding = 1, Suffix = " 单位", Callback = function(v) Hitbox.Size = v end })
task.spawn(function()
    while task.wait(0.5) do
        local ef = getEnemyFolder()
        if ef then
            for _, e in ipairs(ef:GetChildren()) do
                local hd = e:FindFirstChild("Head") local hm = e:FindFirstChildOfClass("Humanoid")
                if hd and hm and hm.Health > 0 then
                    if not originalHeadSizes[hd] then originalHeadSizes[hd] = hd.Size end
                    if Hitbox.Enabled then
                        hd.Size = Vector3.new(Hitbox.Size, Hitbox.Size, Hitbox.Size)
                        hd.CanCollide = false; hd.Transparency = 0.5
                    else
                        hd.Size = originalHeadSizes[hd] or Vector3.new(2,2,1); hd.Transparency = 0
                    end
                end
            end
        end
    end
end)

-- Bhop
local Bhop = {Enabled = false}
local MovementGroup = Tabs.Combat:AddLeftGroupbox("移动", "activity")
MovementGroup:AddToggle("BhopToggle", { Text = "连跳 (按住空格)", Default = false, Callback = function(v) Bhop.Enabled = v end })
RunService.RenderStepped:Connect(function()
    if Bhop.Enabled and UserInputService:IsKeyDown(Enum.KeyCode.Space) and isAlive() then
        local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if hum and hum:GetState() ~= Enum.HumanoidStateType.Jumping and hum:GetState() ~= Enum.HumanoidStateType.Freefall then
            hum.Jump = true
        end
    end
end)

-- ==================== WALLBANG (原始击杀提醒逻辑) ====================
local Wallbang = {Enabled = false, Mode = "自动", Key = Enum.KeyCode.F, KeyHeld = false, Delay = 0.5, HitPart = "身体", SoundEnabled = true, SoundID = "92723765069002"}
local HitNotif = {
    Enabled = true, Style = "胶囊", BgColor = Color3.fromRGB(25,25,25), BgTrans = 0.3,
    TextColor = Color3.fromRGB(255,255,255), DeathColor = Color3.fromRGB(255,50,50),
    OffsetX = 0, OffsetY = 0, Scale = 1, MaxCount = 5, Duration = 4
}
local bgR,bgG,bgB = 25,25,25; local textR,textG,textB = 255,255,255; local deathR,deathG,deathB = 255,50,50

local wallRemote = nil
local wallRemoteFetched = false
local function getWallRemote()
    if wallRemoteFetched then return wallRemote end
    for _, v in next, getgc(true) do
        if type(v) == "table" and rawget(v, string.char(83,104,111,111,116,87,101,97,112,111,110)) then
            wallRemote = v; wallRemoteFetched = true; return v
        end
    end
    return nil
end

local wallCurrentWeapon = nil
local wallLastShot = 0
local wallReloadLock = false
local hitPartMap = {["头部"]="Head", ["身体"]="HumanoidRootPart", ["左腿"]="LeftLowerLeg", ["右腿"]="RightLowerLeg", ["左臂"]="LeftLowerArm", ["右臂"]="RightLowerArm"}
local killSounds = {
    {Name = "超级击杀", ID = "92723765069002"},{Name = "我们之中", ID = "7227567562"},{Name = "怪物杀戮", ID = "132012038491424"},
    {Name = "叮", ID = "2866718318"},{Name = "鲜血", ID = "128741351184513"},{Name = "黄金", ID = "18888511866"},
    {Name = "瓦洛兰特", ID = "18560690982"},{Name = "咚", ID = "7269900245"},{Name = "动漫", ID = "80440627510518"},
    {Name = "现代战争", ID = "130439616552357"},{Name = "战斗", ID = "7228383943"},{Name = "呀", ID = "111609064980370"},
    {Name = "咯", ID = "80847075127412"}
}
local function playSoundSafe(id)
    if not id or id == "" then return end
    local snd = Instance.new("Sound")
    snd.SoundId = "rbxassetid://"..id
    snd.Volume = 1
    snd.Parent = camera
    snd:Play()
    task.delay(3, function() if snd then snd:Destroy() end end)
end

-- Notification system
local notifGui = Instance.new("ScreenGui")
notifGui.Name = "WallbangNotifs"; notifGui.ResetOnSpawn = false; notifGui.IgnoreGuiInset = true
notifGui.Parent = player:WaitForChild("PlayerGui")
local notifTemplate = Instance.new("Frame")
notifTemplate.BackgroundColor3 = HitNotif.BgColor; notifTemplate.BackgroundTransparency = HitNotif.BgTrans
notifTemplate.BorderSizePixel = 0; notifTemplate.Size = UDim2.new(0,200,0,30); notifTemplate.Visible = false
local templateCorner = Instance.new("UICorner"); templateCorner.CornerRadius = UDim.new(0,15); templateCorner.Parent = notifTemplate
local templateLabel = Instance.new("TextLabel")
templateLabel.BackgroundTransparency = 1; templateLabel.Size = UDim2.new(1,-10,1,0); templateLabel.Position = UDim2.new(0,5,0,0)
templateLabel.Font = Enum.Font.Gotham; templateLabel.TextSize = 18; templateLabel.TextColor3 = HitNotif.TextColor
templateLabel.TextStrokeTransparency = 0.8; templateLabel.TextXAlignment = Enum.TextXAlignment.Left; templateLabel.Parent = notifTemplate

local activeNotifs = {}

local function adjustNotifs()
    local baseY = 50 + HitNotif.OffsetY
    local yOff = 0
    for _, entry in ipairs(activeNotifs) do
        local frame = entry.frame
        if frame and frame.Parent then
            local targetX = camera.ViewportSize.X - frame.AbsoluteSize.X - 15 + HitNotif.OffsetX
            local targetY = baseY + yOff
            TweenService:Create(frame, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Position = UDim2.new(0, targetX, 0, targetY)
            }):Play()
            yOff = yOff + frame.AbsoluteSize.Y + 5
        end
    end
end

local function createNotif(text, textColor, bgColor, bgTrans)
    if HitNotif.MaxCount > 0 then
        while #activeNotifs >= HitNotif.MaxCount do
            local old = table.remove(activeNotifs, 1)
            if old.frame and old.frame.Parent then
                old.frame:Destroy()
            end
        end
    end

    local frame = notifTemplate:Clone()
    frame.BackgroundColor3 = bgColor; frame.BackgroundTransparency = 1
    frame.Visible = true; frame.Parent = notifGui
    local label = frame:FindFirstChildOfClass("TextLabel")
    if label then
        label.Text = text; label.TextColor3 = textColor; label.TextSize = 18 * HitNotif.Scale
        label.TextTransparency = 1
    end
    local corner = frame:FindFirstChildOfClass("UICorner")
    if corner then
        corner.CornerRadius = (HitNotif.Style == "胶囊") and UDim.new(0, 15*HitNotif.Scale) or UDim.new(0,0)
    end
    local textSize = TextService:GetTextSize(text, label.TextSize, label.Font, Vector2.new(1920,1080))
    frame.Size = UDim2.new(0, textSize.X + 20*HitNotif.Scale, 0, textSize.Y + 12*HitNotif.Scale)

    frame.Position = UDim2.new(1, 50, 0, 50)
    local entry = {frame = frame, createdAt = tick()}
    table.insert(activeNotifs, entry)

    adjustNotifs()
    TweenService:Create(frame, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        BackgroundTransparency = bgTrans
    }):Play()
    if label then
        TweenService:Create(label, TweenInfo.new(0.25), {TextTransparency = 0}):Play()
    end

    task.delay(HitNotif.Duration, function()
        if frame and frame.Parent then
            TweenService:Create(frame, TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
                BackgroundTransparency = 1,
                Position = frame.Position + UDim2.new(0, 50, 0, 0)
            }):Play()
            task.delay(0.3, function()
                if frame and frame.Parent then frame:Destroy() end
                for i, v in ipairs(activeNotifs) do
                    if v == entry then table.remove(activeNotifs, i); break end
                end
                adjustNotifs()
            end)
        end
    end)
    return entry
end

local function getNearestEnemyWithPart(origin, partName)
    local myTeam
    for _, f in next, CharactersFolder:GetChildren() do
        if f:IsA("Folder") and f:FindFirstChild(player.Name) then myTeam = f.Name break end
    end
    if not myTeam then return nil end
    local nearest, minDist = nil, math.huge
    for _, folder in next, CharactersFolder:GetChildren() do
        if folder:IsA("Folder") and folder.Name ~= myTeam then
            for _, enemy in next, folder:GetChildren() do
                local hum = enemy:FindFirstChildOfClass("Humanoid")
                if hum and hum.Health > 0 then
                    local part = enemy:FindFirstChild(partName) or enemy:FindFirstChild("HumanoidRootPart")
                    if part then
                        local dist = (part.Position - origin).Magnitude
                        if dist < minDist then
                            minDist = dist
                            nearest = {pos = part.Position, part = part, model = enemy, hum = hum}
                        end
                    end
                end
            end
        end
    end
    return nearest
end

local function isWallbangWeapon(t)
    if not t then return false end
    local p = rawget(t, "Properties")
    return p and rawget(p, "FireRate") and rawget(p, "BulletsPerShot") and rawget(p, "Rounds")
end

local function updateWallWeapon()
    for _, v in next, getgc(true) do
        if type(v) == "table" and rawget(v, "IsEquipped") and rawget(v, "Identifier") and rawget(v, "Player") == player then
            if isWallbangWeapon(v) then wallCurrentWeapon = v; return true end
        end
    end
    wallCurrentWeapon = nil; return false
end

local function wallReload()
    if not wallCurrentWeapon or wallReloadLock then return end
    local prop = rawget(wallCurrentWeapon, "Properties")
    local max = rawget(prop, "Rounds"); local cur = rawget(wallCurrentWeapon, "Rounds"); local cap = rawget(wallCurrentWeapon, "Capacity")
    if not (max and cur and cap) then return end
    if cur < max and cap > 0 then
        wallReloadLock = true
        local need = math.min(max - cur, cap)
        wallCurrentWeapon.Rounds = cur + need
        wallCurrentWeapon.Capacity = cap - need
        task.wait(0.05)
        wallReloadLock = false
    end
end

local function wallbangShoot()
    if not Wallbang.Enabled then return end
    task.wait(math.random() * 0.05)
    if tick() - wallLastShot < Wallbang.Delay then return end
    local char = player.Character
    if not char or char:GetAttribute("Dead") then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if not wallCurrentWeapon or not rawget(wallCurrentWeapon, "IsEquipped") then
        if not updateWallWeapon() then return end
    end
    if not isWallbangWeapon(wallCurrentWeapon) then wallCurrentWeapon = nil; return end
    local prop = rawget(wallCurrentWeapon, "Properties")
    local max = rawget(prop, "Rounds"); local cur = rawget(wallCurrentWeapon, "Rounds")
    if not (max and cur) then return end
    if cur < max * 0.3 then wallReload() end
    if cur <= 0 then wallReload(); return end
    local remote = getWallRemote()
    if not remote or not remote.ShootWeapon then return end
    local partName = hitPartMap[Wallbang.HitPart] or "HumanoidRootPart"
    local target = getNearestEnemyWithPart(hrp.Position, partName)
    if not target then return end
    wallLastShot = tick()
    local origin = camera.CFrame.Position
    local dir = (target.pos - origin).Unit
    wallCurrentWeapon.Rounds = cur - 1
    remote.ShootWeapon.Send({
        IsSniperScoped = false, ShootingHand = "Right",
        Identifier = wallCurrentWeapon.Identifier, Capacity = wallCurrentWeapon.Capacity, Rounds = wallCurrentWeapon.Rounds,
        Bullets = {{ Direction = dir, Origin = origin, Hits = {{ Instance = target.part, Position = target.pos, Normal = -dir, Material = "Plastic", Distance = (target.pos - origin).Magnitude, Exit = false }} }}
    })
    local enemyName = target.model and target.model.Name or "Unknown"
    if HitNotif.Enabled then
        createNotif(string.format("Attacking %s - %s - HP: %d", enemyName, Wallbang.HitPart, math.floor(target.hum.Health)), HitNotif.TextColor, HitNotif.BgColor, HitNotif.BgTrans)
    end
    local targetHum = target.hum
    local targetName = enemyName
    task.delay(0.3, function()
        local isDead = false
        if targetHum then
            if targetHum.Health <= 0 or not targetHum.Parent then isDead = true end
        else
            isDead = true
        end
        if isDead then
            if HitNotif.Enabled then
                createNotif(string.format("Attacking %s - %s - DEAD", targetName, Wallbang.HitPart), HitNotif.DeathColor, HitNotif.BgColor, HitNotif.BgTrans)
            end
            if Wallbang.SoundEnabled then playSoundSafe(Wallbang.SoundID) end
        end
    end)
end

UserInputService.InputBegan:Connect(function(i) if i.UserInputType == Wallbang.Key or i.KeyCode == Wallbang.Key then Wallbang.KeyHeld = true end end)
UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Wallbang.Key or i.KeyCode == Wallbang.Key then Wallbang.KeyHeld = false end end)
RunService.Heartbeat:Connect(function()
    if not Wallbang.Enabled then return end
    local shoot = false
    if Wallbang.Mode == "自动" then shoot = true elseif Wallbang.Mode == "热键" then shoot = Wallbang.KeyHeld end
    if shoot and isAlive() then wallbangShoot() end
end)
task.spawn(function()
    while task.wait(0.12) do
        if Wallbang.Enabled and wallCurrentWeapon and isWallbangWeapon(wallCurrentWeapon) then
            local cur = rawget(wallCurrentWeapon, "Rounds"); local prop = rawget(wallCurrentWeapon, "Properties")
            if prop and cur then
                local max = rawget(prop, "Rounds")
                if max and cur < max then wallReload() end
            end
        end
    end
end)

local WallbangGroup = Tabs.Combat:AddRightGroupbox("静默穿墙", "crosshair")
WallbangGroup:AddToggle("WallbangToggle", { Text = "启用穿墙", Default = false, Callback = function(v) Wallbang.Enabled = v end })
WallbangGroup:AddDropdown("WallbangMode", { Text = "模式", Values = {"自动","热键"}, Default = "自动", Callback = function(v) Wallbang.Mode = v end })
WallbangGroup:AddDropdown("WallbangHotkey", { Text = "热键", Values = hotkeyValues, Default = getHotkeyName(Wallbang.Key), Callback = function(v) Wallbang.Key = hotkeyMap[v] or Enum.KeyCode.F end })
WallbangGroup:AddSlider("WallbangDelay", { Text = "射击间隔", Default = 0.5, Min = 0.2, Max = 2, Rounding = 2, Suffix = "秒", Callback = function(v) Wallbang.Delay = v end })
WallbangGroup:AddDropdown("WallbangHitPart", { Text = "击打部位", Values = {"头部","身体","左腿","右腿","左臂","右臂"}, Default = "身体", Callback = function(v) Wallbang.HitPart = v end })
WallbangGroup:AddToggle("WallbangSoundToggle", { Text = "击杀音效", Default = true, Callback = function(v) Wallbang.SoundEnabled = v end })
WallbangGroup:AddDropdown("WallbangSound", { Text = "击杀音效", Values = {"超级击杀","我们之中","怪物杀戮","叮","鲜血","黄金","瓦洛兰特","咚","动漫","现代战争","战斗","呀","咯"}, Default = "超级击杀", Callback = function(v) for _,s in ipairs(killSounds) do if s.Name==v then Wallbang.SoundID=s.ID; playSoundSafe(s.ID) break end end end })
WallbangGroup:AddDivider(); WallbangGroup:AddLabel("击杀提示")
WallbangGroup:AddToggle("HitNotifEnabled", { Text = "启用击杀提示", Default = true, Callback = function(v) HitNotif.Enabled = v end })
WallbangGroup:AddDropdown("HitNotifStyle", { Text = "样式", Values = {"胶囊","矩形"}, Default = "胶囊", Callback = function(v) HitNotif.Style = v end })
WallbangGroup:AddSlider("HitNotifBgTrans", { Text = "背景透明度", Default = 0.3, Min = 0, Max = 1, Rounding = 1, Suffix = "", Callback = function(v) HitNotif.BgTrans = v end })
WallbangGroup:AddSlider("HitNotifOffsetX", { Text = "X偏移", Default = 0, Min = -500, Max = 500, Rounding = 0, Suffix = "px", Callback = function(v) HitNotif.OffsetX = v end })
WallbangGroup:AddSlider("HitNotifOffsetY", { Text = "Y偏移", Default = 0, Min = -500, Max = 500, Rounding = 0, Suffix = "px", Callback = function(v) HitNotif.OffsetY = v end })
WallbangGroup:AddSlider("HitNotifScale", { Text = "整体大小", Default = 1, Min = 0.5, Max = 2, Rounding = 1, Suffix = "x", Callback = function(v) HitNotif.Scale = v end })
WallbangGroup:AddSlider("NotifMaxCount", { Text = "最大通知数量", Default = 5, Min = 0, Max = 15, Rounding = 0, Suffix = " (0=无限)", Callback = function(v) HitNotif.MaxCount = v end })
WallbangGroup:AddSlider("NotifDuration", { Text = "通知停留时间", Default = 4, Min = 1, Max = 10, Rounding = 1, Suffix = "秒", Callback = function(v) HitNotif.Duration = v end })
WallbangGroup:AddLabel("背景颜色 RGB")
WallbangGroup:AddSlider("BgR", { Text = "红", Default = 25, Min = 0, Max = 255, Rounding = 0, Callback = function(v) bgR=v; HitNotif.BgColor=Color3.fromRGB(bgR,bgG,bgB) end })
WallbangGroup:AddSlider("BgG", { Text = "绿", Default = 25, Min = 0, Max = 255, Rounding = 0, Callback = function(v) bgG=v; HitNotif.BgColor=Color3.fromRGB(bgR,bgG,bgB) end })
WallbangGroup:AddSlider("BgB", { Text = "蓝", Default = 25, Min = 0, Max = 255, Rounding = 0, Callback = function(v) bgB=v; HitNotif.BgColor=Color3.fromRGB(bgR,bgG,bgB) end })
WallbangGroup:AddLabel("文字颜色 RGB")
WallbangGroup:AddSlider("TextR", { Text = "红", Default = 255, Min = 0, Max = 255, Rounding = 0, Callback = function(v) textR=v; HitNotif.TextColor=Color3.fromRGB(textR,textG,textB) end })
WallbangGroup:AddSlider("TextG", { Text = "绿", Default = 255, Min = 0, Max = 255, Rounding = 0, Callback = function(v) textG=v; HitNotif.TextColor=Color3.fromRGB(textR,textG,textB) end })
WallbangGroup:AddSlider("TextB", { Text = "蓝", Default = 255, Min = 0, Max = 255, Rounding = 0, Callback = function(v) textB=v; HitNotif.TextColor=Color3.fromRGB(textR,textG,textB) end })
WallbangGroup:AddLabel("击杀文字颜色 RGB")
WallbangGroup:AddSlider("DeathR", { Text = "红", Default = 255, Min = 0, Max = 255, Rounding = 0, Callback = function(v) deathR=v; HitNotif.DeathColor=Color3.fromRGB(deathR,deathG,deathB) end })
WallbangGroup:AddSlider("DeathG", { Text = "绿", Default = 50, Min = 0, Max = 255, Rounding = 0, Callback = function(v) deathG=v; HitNotif.DeathColor=Color3.fromRGB(deathR,deathG,deathB) end })
WallbangGroup:AddSlider("DeathB", { Text = "蓝", Default = 50, Min = 0, Max = 255, Rounding = 0, Callback = function(v) deathB=v; HitNotif.DeathColor=Color3.fromRGB(deathR,deathG,deathB) end })

-- ==================== SILENT AIM (队伍检查已修复) ====================
local SilentAimSettings = {
    Enabled = false, ToggleKey = "RightAlt", TeamCheck = false, VisibleCheck = false,
    TargetPart = "HumanoidRootPart", SilentAimMethod = "Raycast", FOVRadius = 130,
    FOVVisible = true, ShowSilentAimTarget = false, ShowTracer = false,
    MouseHitPrediction = false, MouseHitPredictionAmount = 0.165, HitChance = 100,
    FixedFOV = true, TargetIndicatorRadius = 20, IndicatorRotationEnabled = false,
    IndicatorRotationSpeed = 1, IndicatorRainbowEnabled = false, IndicatorRainbowSpeed = 1,
    MaxDistance = 500, Tracer_Y_Offset = 0, PriorityMode = "准星最近",
    TargetInfoStyle = "面板", ShowTargetName = false, ShowTargetHealth = false,
    ShowTargetDistance = false, ShowTargetCategory = false, ShowDamageNotifier = false,
    IndependentPanelPosition = "200,200", IndependentPanelPinned = false,
    LeakAndHitMode = false, Wallbang = false, EnableNameTargeting = false,
    TargetName1 = "", TargetName2 = "", TargetName3 = "", TargetMode = "玩家"
}
local ValidTargetParts = {"Head", "HumanoidRootPart"}
local PredictionAmount = 0.165
local currentTargetPart = nil
local currentRotationAngle = 0
local currentIndicatorHue = 0
local npcList = {}
local targetMap = {}
local lockedTargetCharacter = nil

-- Drawing objects (Silent Aim)
local target_indicator_circle = Drawing.new("Circle")
target_indicator_circle.Visible = false; target_indicator_circle.ZIndex = 1000; target_indicator_circle.Thickness = 2; target_indicator_circle.Filled = false
local target_indicator_lines = {}
for i = 1, 5 do local line = Drawing.new("Line"); line.Visible = false; line.ZIndex = 1000; line.Thickness = 2; table.insert(target_indicator_lines, line) end
local tracer_line = Drawing.new("Line")
tracer_line.Visible = false; tracer_line.ZIndex = 998; tracer_line.Color = Color3.fromRGB(255, 255, 0); tracer_line.Thickness = 1; tracer_line.Transparency = 1

local overhead_info_texts = {
    Name = Drawing.new("Text"), Health = Drawing.new("Text"), Distance = Drawing.new("Text"), Category = Drawing.new("Text")
}
for _, text in pairs(overhead_info_texts) do
    text.Visible = false; text.ZIndex = 1001; text.Font = Drawing.Fonts.Plex; text.Size = 14; text.Color = Color3.fromRGB(255, 255, 255); text.Center = true; text.Outline = true
end

local panel_info_bg = Drawing.new("Square")
panel_info_bg.Visible = false; panel_info_bg.ZIndex = 1002; panel_info_bg.Color = Color3.fromRGB(0, 0, 0); panel_info_bg.Thickness = 0; panel_info_bg.Filled = true; panel_info_bg.Transparency = 0.5
local panel_info_texts = {
    Name = Drawing.new("Text"), Health = Drawing.new("Text"), Distance = Drawing.new("Text"), Category = Drawing.new("Text")
}
for _, text in pairs(panel_info_texts) do
    text.Visible = false; text.ZIndex = 1003; text.Font = Drawing.Fonts.Plex; text.Size = 14; text.Color = Color3.fromRGB(255, 255, 255); text.Center = false; text.Outline = true
end

local FOVCircleGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
FOVCircleGui.Name = "SilentFOVGui"; FOVCircleGui.ResetOnSpawn = false; FOVCircleGui.IgnoreGuiInset = true
local FOVCircleFrame = Instance.new("Frame", FOVCircleGui)
FOVCircleFrame.AnchorPoint = Vector2.new(0.5, 0.5); FOVCircleFrame.Position = UDim2.fromScale(0.5, 0.5); FOVCircleFrame.BackgroundTransparency = 1
local FOVStroke = Instance.new("UIStroke", FOVCircleFrame); FOVStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border; FOVStroke.Thickness = 1
local FOVCorner = Instance.new("UICorner", FOVCircleFrame); FOVCorner.CornerRadius = UDim.new(1, 0)

local IndependentPanelGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
IndependentPanelGui.Name = "IndependentPanelGui"; IndependentPanelGui.ResetOnSpawn = false
local IndependentPanelFrame = Instance.new("Frame", IndependentPanelGui)
IndependentPanelFrame.Size = UDim2.fromOffset(160, 100); IndependentPanelFrame.BackgroundColor3 = Color3.fromRGB(30,30,30)
IndependentPanelFrame.BackgroundTransparency = 0.3; IndependentPanelFrame.BorderSizePixel = 1; IndependentPanelFrame.Visible = false; IndependentPanelFrame.Active = true
local IPCorner = Instance.new("UICorner", IndependentPanelFrame); IPCorner.CornerRadius = UDim.new(0,4)
local IPListLayout = Instance.new("UIListLayout", IndependentPanelFrame); IPListLayout.Padding = UDim.new(0,5)
local independent_panel_texts = {}
for i, name in ipairs({"Name", "Health", "Distance", "Category"}) do
    local label = Instance.new("TextLabel", IndependentPanelFrame)
    label.Name = name; label.Size = UDim2.new(1, -10, 0, 15); label.BackgroundTransparency = 1
    label.Font = Enum.Font.SourceSans; label.TextSize = 14; label.TextColor3 = Color3.new(1,1,1); label.TextXAlignment = Enum.TextXAlignment.Left; label.LayoutOrder = i
    independent_panel_texts[name] = label
end
IndependentPanelFrame.InputBegan:Connect(function(input) if input.UserInputType == Enum.UserInputType.MouseButton1 and IndependentPanelFrame.Draggable then IndependentPanelFrame.Position = UDim2.fromOffset(UserInputService:GetMouseLocation().X, UserInputService:GetMouseLocation().Y) end end)

-- 队伍检查修复：基于文件夹判断敌我
local function IsEnemy(character)
    if not character or not player.Character then return true end
    local playerFolder = player.Character.Parent
    local targetFolder = character.Parent
    if playerFolder and targetFolder and (playerFolder.Name == "Terrorists" or playerFolder.Name == "Counter-Terrorists") and (targetFolder.Name == "Terrorists" or targetFolder.Name == "Counter-Terrorists") then
        return playerFolder ~= targetFolder
    end
    return true
end

local ExpectedArguments = {
    FindPartOnRayWithIgnoreList = { ArgCountRequired = 3, Args = {"Instance", "Ray", "table", "boolean", "boolean"} },
    FindPartOnRayWithWhitelist = { ArgCountRequired = 3, Args = {"Instance", "Ray", "table", "boolean"} },
    FindPartOnRay = { ArgCountRequired = 2, Args = {"Instance", "Ray", "Instance", "boolean", "boolean"} },
    Raycast = { ArgCountRequired = 3, Args = {"Instance", "Vector3", "Vector3", "RaycastParams"} }
}
function CalculateChance(Percentage) return math.random() <= (Percentage/100) end
function ValidateArguments(Args, RayMethod)
    local m = 0
    if #Args < RayMethod.ArgCountRequired then return false end
    for i, arg in next, Args do if typeof(arg) == RayMethod.Args[i] then m = m+1 end end
    return m >= RayMethod.ArgCountRequired
end
function getDirection(Origin, Position) return (Position-Origin).Unit*1000 end
function isNPC(obj) return obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj.Humanoid.Health>0 and obj:FindFirstChild("HumanoidRootPart") and not Players:GetPlayerFromCharacter(obj) end
function getTargetCategory(character)
    if not character then return "无" end
    if Players:GetPlayerFromCharacter(character) then return "玩家" end
    if SilentAimSettings.EnableNameTargeting then
        local name = character.Name:lower()
        local t1,t2,t3 = SilentAimSettings.TargetName1:lower(), SilentAimSettings.TargetName2:lower(), SilentAimSettings.TargetName3:lower()
        if (t1~="" and string.find(name, t1,1,true)) or (t2~="" and string.find(name, t2,1,true)) or (t3~="" and string.find(name, t3,1,true)) then return "添加的" end
    end
    if character:FindFirstChild("Humanoid") then return "NPC" end
    return "未知"
end
function updateNPCs()
    local new = {}; local added = {}
    if SilentAimSettings.EnableNameTargeting then
        local subs = {}
        if SilentAimSettings.TargetName1~="" then table.insert(subs, SilentAimSettings.TargetName1:lower()) end
        if SilentAimSettings.TargetName2~="" then table.insert(subs, SilentAimSettings.TargetName2:lower()) end
        if SilentAimSettings.TargetName3~="" then table.insert(subs, SilentAimSettings.TargetName3:lower()) end
        if #subs>0 then
            for _,m in ipairs(workspace:GetDescendants()) do
                if isNPC(m) then
                    for _,sub in ipairs(subs) do
                        if string.find(m.Name:lower(), sub,1,true) and not added[m] then
                            table.insert(new, m); added[m]=true; break
                        end
                    end
                end
            end
        end
    end
    for _,v in ipairs(workspace:GetChildren()) do
        if isNPC(v) and not added[v] then table.insert(new, v); added[v]=true end
    end
    npcList = new
end
local function isPartVisible(part, customOrigin)
    if not part then return false end
    local char = player.Character; if not char then return false end
    local origin = customOrigin or camera.CFrame.Position
    local dir = part.Position - origin
    local params = RaycastParams.new(); params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {char, part.Parent}
    local result = workspace:Raycast(origin, dir.Unit*dir.Magnitude, params)
    return not result
end
local function getPositionOnScreen(Vector)
    local v, on = camera:WorldToViewportPoint(Vector)
    return Vector2.new(v.X, v.Y), on
end

local function getClosestPlayer()
    local char = player.Character; if not char or not char:FindFirstChild("HumanoidRootPart") then return nil end
    local root = char.HumanoidRootPart
    local aimPoint = SilentAimSettings.FixedFOV and (camera.ViewportSize/2) or UserInputService:GetMouseLocation()
    local candidates = {}
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= player then
            local chr = plr.Character
            local hum = chr and chr:FindFirstChildOfClass("Humanoid")
            if chr and chr:FindFirstChild("HumanoidRootPart") and hum and hum.Health>0 then
                if not (SilentAimSettings.TeamCheck and not IsEnemy(chr)) then
                    if not (SilentAimSettings.VisibleCheck and not isPartVisible(chr.HumanoidRootPart, char.Head.Position)) then
                        local dist = (root.Position - chr.HumanoidRootPart.Position).Magnitude
                        if dist <= SilentAimSettings.MaxDistance then
                            local part = chr:FindFirstChild(SilentAimSettings.TargetPart) or chr.HumanoidRootPart
                            if SilentAimSettings.TargetPart == "Random" then part = chr[ValidTargetParts[math.random(1,2)]] end
                            if part then
                                local scr, on = getPositionOnScreen(part.Position)
                                if on then
                                    local fov = (aimPoint - scr).Magnitude
                                    if fov <= SilentAimSettings.FOVRadius then
                                        table.insert(candidates, {character=chr, fov=fov, dist=dist, health=hum.Health})
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    if #candidates==0 then return nil end
    table.sort(candidates, function(a,b)
        if SilentAimSettings.PriorityMode=="最低血量" then return a.health<b.health
        elseif SilentAimSettings.PriorityMode=="距离最近" then return a.dist<b.dist
        else return a.fov<b.fov end
    end)
    return candidates[1].character
end

local function getNPCTarget()
    local char = player.Character; if not char or not char:FindFirstChild("HumanoidRootPart") then return nil end
    local root = char.HumanoidRootPart
    local aimPoint = SilentAimSettings.FixedFOV and (camera.ViewportSize/2) or UserInputService:GetMouseLocation()
    local candidates = {}
    for _, npc in ipairs(npcList) do
        if not (SilentAimSettings.TeamCheck and not IsEnemy(npc)) then
            local hum = npc:FindFirstChildOfClass("Humanoid")
            if npc and npc.PrimaryPart and hum and hum.Health>0 then
                if not (SilentAimSettings.VisibleCheck and not isPartVisible(npc.PrimaryPart, char.Head.Position)) then
                    local dist = (root.Position - npc.PrimaryPart.Position).Magnitude
                    if dist <= SilentAimSettings.MaxDistance then
                        local part = npc:FindFirstChild(SilentAimSettings.TargetPart) or npc.PrimaryPart
                        if SilentAimSettings.TargetPart == "Random" then part = npc[ValidTargetParts[math.random(1,2)]] end
                        if part then
                            local scr, on = getPositionOnScreen(part.Position)
                            if on then
                                local fov = (aimPoint - scr).Magnitude
                                if fov <= SilentAimSettings.FOVRadius then
                                    table.insert(candidates, {character=npc, fov=fov, dist=dist, health=hum.Health})
                                end
                            end
                        end
                    end
                end
            end
        end
    end
    if #candidates==0 then return nil end
    table.sort(candidates, function(a,b)
        if SilentAimSettings.PriorityMode=="最低血量" then return a.health<b.health
        elseif SilentAimSettings.PriorityMode=="距离最近" then return a.dist<b.dist
        else return a.fov<b.fov end
    end)
    return candidates[1].character
end

function getPolygonPoints(center, radius, sides)
    local points = {}
    local rot = SilentAimSettings.IndicatorRotationEnabled and currentRotationAngle or 0
    for i=1,sides do
        local angle = (i-1)*(2*math.pi/sides) - math.pi/2 + rot
        table.insert(points, Vector2.new(center.X+radius*math.cos(angle), center.Y+radius*math.sin(angle)))
    end
    return points
end

function hideAllVisuals()
    target_indicator_circle.Visible = false
    for _,l in ipairs(target_indicator_lines) do l.Visible=false end
    for _,t in pairs(overhead_info_texts) do t.Visible=false end
    panel_info_bg.Visible = false
    for _,t in pairs(panel_info_texts) do t.Visible=false end
    if IndependentPanelFrame then IndependentPanelFrame.Visible = false end
end

local lastHealthValues = {}
local damageIndicators = {}
local DAMAGE_INDICATOR_FADE_TIME = 1

coroutine.resume(coroutine.create(function()
    RunService.RenderStepped:Connect(function()
        if SilentAimSettings.IndicatorRotationEnabled then currentRotationAngle = (currentRotationAngle + SilentAimSettings.IndicatorRotationSpeed/50) % (2*math.pi) end
        if SilentAimSettings.IndicatorRainbowEnabled then currentIndicatorHue = (currentIndicatorHue + SilentAimSettings.IndicatorRainbowSpeed/200) % 1 end
        
        local isEnabled = Toggles.SilentEnabledToggle.Value
        currentTargetPart = nil
        local currentTargetCharacter = nil

        if isEnabled then
            if lockedTargetCharacter then
                currentTargetCharacter = lockedTargetCharacter
            else
                local targetMode = SilentAimSettings.TargetMode
                local playerTarget, npcTarget
                if targetMode=="玩家" or targetMode=="所有" then playerTarget = getClosestPlayer() end
                if targetMode=="NPC" or targetMode=="所有" then npcTarget = getNPCTarget() end
                if playerTarget and npcTarget then
                    if SilentAimSettings.PriorityMode=="最低血量" then
                        local ph = playerTarget:FindFirstChildOfClass("Humanoid")
                        local nh = npcTarget:FindFirstChildOfClass("Humanoid")
                        currentTargetCharacter = (ph and nh and ph.Health<=nh.Health) and playerTarget or npcTarget
                    else
                        local pDist = (player.Character.HumanoidRootPart.Position - playerTarget.HumanoidRootPart.Position).Magnitude
                        local nDist = (player.Character.HumanoidRootPart.Position - npcTarget.HumanoidRootPart.Position).Magnitude
                        currentTargetCharacter = pDist < nDist and playerTarget or npcTarget
                    end
                else
                    currentTargetCharacter = playerTarget or npcTarget
                end
            end
        end

        if currentTargetCharacter then
            local hum = currentTargetCharacter:FindFirstChildOfClass("Humanoid")
            if not hum or hum.Health<=0 then
                if lockedTargetCharacter == currentTargetCharacter then lockedTargetCharacter = nil end
                currentTargetCharacter = nil
            else
                if SilentAimSettings.LeakAndHitMode then
                    for _,p in ipairs(currentTargetCharacter:GetDescendants()) do
                        if p:IsA("BasePart") and p.Parent == currentTargetCharacter then
                            if isPartVisible(p) then currentTargetPart = p; break end
                        end
                    end
                else
                    local name = SilentAimSettings.TargetPart
                    if name=="Random" then currentTargetPart = currentTargetCharacter[ValidTargetParts[math.random(1,2)]]
                    else currentTargetPart = currentTargetCharacter:FindFirstChild(name) or currentTargetCharacter:FindFirstChild("HumanoidRootPart") end
                end
            end
        end

        if isEnabled and currentTargetPart and SilentAimSettings.ShowDamageNotifier then
            local hum = currentTargetPart.Parent:FindFirstChildOfClass("Humanoid")
            if hum then
                local ch = hum.Health
                local last = lastHealthValues[hum]
                if last and ch < last then
                    local dmg = math.floor(last-ch)
                    if dmg>0 then
                        local ind = {Created=tick(), Position=getPositionOnScreen(currentTargetPart.Position), TextObject=Drawing.new("Text")}
                        ind.TextObject.Font=Drawing.Fonts.Monospace; ind.TextObject.Text="-"..dmg; ind.TextObject.Color=Color3.fromRGB(255,50,50); ind.TextObject.Size=20; ind.TextObject.Center=true; ind.TextObject.Outline=true
                        table.insert(damageIndicators, ind)
                    end
                end
                lastHealthValues[hum] = ch
            end
        end
        for i=#damageIndicators,1,-1 do
            local ind = damageIndicators[i]; local age = tick()-ind.Created
            if age > DAMAGE_INDICATOR_FADE_TIME then ind.TextObject:Remove(); table.remove(damageIndicators,i)
            else
                local prog = age/DAMAGE_INDICATOR_FADE_TIME
                ind.TextObject.Position = ind.Position - Vector2.new(0, prog*40)
                ind.TextObject.Transparency = prog; ind.TextObject.Visible = true
            end
        end

        hideAllVisuals()
        
        if isEnabled and currentTargetPart then
            local rootScreen, onScreen = getPositionOnScreen(currentTargetPart.Position)

            if onScreen and Toggles.ShowTargetToggle.Value then
                local radius = SilentAimSettings.TargetIndicatorRadius
                local style = Options.IndicatorStyleDropdown.Value
                local col; local vis = isPartVisible(currentTargetPart)
                if vis then col = Color3.fromRGB(0,255,0); radius = radius*0.6
                elseif SilentAimSettings.IndicatorRainbowEnabled then col = Color3.fromHSV(currentIndicatorHue,1,1)
                else col = Options.TargetIndicatorColorPicker.Value end
                if style=="Circle" then
                    target_indicator_circle.Visible=true; target_indicator_circle.Color=col; target_indicator_circle.Radius=radius; target_indicator_circle.Position=rootScreen
                elseif style=="Triangle" then
                    local pts = getPolygonPoints(rootScreen, radius, 3)
                    for i=1,3 do local l=target_indicator_lines[i]; l.Visible=true; l.Color=col; l.From=pts[i]; l.To=pts[i%3+1] end
                elseif style=="Pentagram" then
                    local pts = getPolygonPoints(rootScreen, radius, 5)
                    local order = {1,3,5,2,4}
                    for i=1,5 do local l=target_indicator_lines[i]; l.Visible=true; l.Color=col; l.From=pts[order[i]]; l.To=pts[order[i%5+1]] end
                end
            end

            local showInfo = Toggles.ShowTargetNameToggle.Value or Toggles.ShowTargetHealthToggle.Value or Toggles.ShowTargetDistanceToggle.Value or Toggles.ShowTargetCategoryToggle.Value
            if showInfo then
                local targetChar = currentTargetPart.Parent
                local hum = targetChar:FindFirstChildOfClass("Humanoid")
                local plr = Players:GetPlayerFromCharacter(targetChar)
                local localRoot = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                if hum and localRoot then
                    local name = plr and plr.Name or targetChar.Name
                    local hp = math.floor(hum.Health)
                    local dist = math.floor((localRoot.Position - currentTargetPart.Position).Magnitude)
                    local cat = getTargetCategory(targetChar)
                    local style = SilentAimSettings.TargetInfoStyle
                    if style=="独立面板" then
                        IndependentPanelFrame.Visible = true
                        independent_panel_texts.Name.Visible = Toggles.ShowTargetNameToggle.Value
                        independent_panel_texts.Health.Visible = Toggles.ShowTargetHealthToggle.Value
                        independent_panel_texts.Distance.Visible = Toggles.ShowTargetDistanceToggle.Value
                        independent_panel_texts.Category.Visible = Toggles.ShowTargetCategoryToggle.Value
                        if Toggles.ShowTargetNameToggle.Value then independent_panel_texts.Name.Text="目标: "..name end
                        if Toggles.ShowTargetHealthToggle.Value then independent_panel_texts.Health.Text="血量: "..hp end
                        if Toggles.ShowTargetDistanceToggle.Value then independent_panel_texts.Distance.Text="距离: "..dist.."m" end
                        if Toggles.ShowTargetCategoryToggle.Value then independent_panel_texts.Category.Text="类别: "..cat end
                    elseif style=="面板" and onScreen then
                        local rad = SilentAimSettings.TargetIndicatorRadius
                        local lines = 0; local lh = 15; local base = rootScreen + Vector2.new(rad+5, -22)
                        if Toggles.ShowTargetNameToggle.Value then local t=panel_info_texts.Name; t.Text=name; t.Position=base+Vector2.new(5,5+lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetHealthToggle.Value then local t=panel_info_texts.Health; t.Text="血量: "..hp; t.Position=base+Vector2.new(5,5+lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetDistanceToggle.Value then local t=panel_info_texts.Distance; t.Text="距离: "..dist.."m"; t.Position=base+Vector2.new(5,5+lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetCategoryToggle.Value then local t=panel_info_texts.Category; t.Text="类别: "..cat; t.Position=base+Vector2.new(5,5+lines*lh); t.Visible=true; lines=lines+1 end
                        if lines>0 then panel_info_bg.Position=base; panel_info_bg.Size=Vector2.new(120, 10+lines*lh); panel_info_bg.Visible=true end
                    elseif style=="头顶" and onScreen then
                        local rad = SilentAimSettings.TargetIndicatorRadius
                        local lines = 0; local lh = 15; local baseY = rootScreen.Y - rad - 10
                        if Toggles.ShowTargetNameToggle.Value then local t=overhead_info_texts.Name; t.Text="["..name.."]"; t.Position=Vector2.new(rootScreen.X, baseY-lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetHealthToggle.Value then local t=overhead_info_texts.Health; t.Text="["..hp.."]"; t.Position=Vector2.new(rootScreen.X, baseY-lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetDistanceToggle.Value then local t=overhead_info_texts.Distance; t.Text="["..dist.."m]"; t.Position=Vector2.new(rootScreen.X, baseY-lines*lh); t.Visible=true; lines=lines+1 end
                        if Toggles.ShowTargetCategoryToggle.Value then local t=overhead_info_texts.Category; t.Text="["..cat.."]"; t.Position=Vector2.new(rootScreen.X, baseY-lines*lh); t.Visible=true; lines=lines+1 end
                    end
                end
            end
        elseif isEnabled and SilentAimSettings.TargetInfoStyle=="独立面板" then
            IndependentPanelFrame.Visible = true
            independent_panel_texts.Name.Visible = true; independent_panel_texts.Health.Visible = true
            independent_panel_texts.Distance.Visible = false; independent_panel_texts.Category.Visible = false
            independent_panel_texts.Name.Text = "状态: 自动索敌中..."; independent_panel_texts.Health.Text = "目标: 无"
        end

        if Toggles.ShowTracerToggle.Value and isEnabled and currentTargetPart then
            local targetHead = currentTargetPart.Parent:FindFirstChild("Head")
            local tracerPos = (targetHead and targetHead.Position) or currentTargetPart.Position
            tracerPos = tracerPos - Vector3.new(0, SilentAimSettings.Tracer_Y_Offset, 0)
            local scr, on = getPositionOnScreen(tracerPos)
            tracer_line.Visible = on
            if on then tracer_line.From = camera.ViewportSize/2; tracer_line.To = scr; tracer_line.Color = Options.TracerColorPicker.Value end
        else tracer_line.Visible = false end
        
        if Toggles.FOVVisibleToggle.Value then
            if Toggles.FixedFOVToggle.Value then FOVCircleFrame.Position = UDim2.fromScale(0.5,0.5) else FOVCircleFrame.Position = UDim2.fromOffset(UserInputService:GetMouseLocation().X, UserInputService:GetMouseLocation().Y) end
        end
    end)
end))

-- Silent Aim Hooks
local hooksInstalled = false
local function installSilentHooks()
    if hooksInstalled then return end
    hooksInstalled = true
    local oldNamecall
    oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(...)
        local Method = getnamecallmethod()
        local Args = {...}
        local self = Args[1]
        if Toggles.SilentEnabledToggle.Value and not checkcaller() and CalculateChance(SilentAimSettings.HitChance) and currentTargetPart then
            local cm = SilentAimSettings.SilentAimMethod
            if Method == "FindPartOnRayWithIgnoreList" and cm == Method and ValidateArguments(Args, ExpectedArguments.FindPartOnRayWithIgnoreList) then
                if SilentAimSettings.Wallbang then return currentTargetPart, currentTargetPart.Position, currentTargetPart.CFrame.LookVector, currentTargetPart.Material end
                local ray = Args[2]; Args[2] = Ray.new(ray.Origin, getDirection(ray.Origin, currentTargetPart.Position))
                return oldNamecall(unpack(Args))
            elseif Method == "Raycast" and cm == Method and ValidateArguments(Args, ExpectedArguments.Raycast) then
                if SilentAimSettings.Wallbang then
                    local origin = Args[2]; local dir = getDirection(origin, currentTargetPart.Position)
                    local wp = RaycastParams.new(); wp.FilterType = Enum.RaycastFilterType.Include; wp.FilterDescendantsInstances = {currentTargetPart.Parent}
                    return oldNamecall(self, origin, dir, wp)
                end
                Args[3] = getDirection(Args[2], currentTargetPart.Position)
                return oldNamecall(unpack(Args))
            elseif Method == "FindPartOnRayWithWhitelist" and cm == Method and ValidateArguments(Args, ExpectedArguments.FindPartOnRayWithWhitelist) then
                if SilentAimSettings.Wallbang then return currentTargetPart, currentTargetPart.Position, currentTargetPart.CFrame.LookVector, currentTargetPart.Material end
                local ray = Args[2]; Args[2] = Ray.new(ray.Origin, getDirection(ray.Origin, currentTargetPart.Position))
                return oldNamecall(unpack(Args))
            elseif (Method == "FindPartOnRay" or Method == "findPartOnRay") and cm:lower() == Method:lower() and ValidateArguments(Args, ExpectedArguments.FindPartOnRay) then
                if SilentAimSettings.Wallbang then return currentTargetPart, currentTargetPart.Position, currentTargetPart.CFrame.LookVector, currentTargetPart.Material end
                local ray = Args[2]; Args[2] = Ray.new(ray.Origin, getDirection(ray.Origin, currentTargetPart.Position))
                return oldNamecall(unpack(Args))
            elseif (Method == "ScreenPointToRay" or Method == "ViewportPointToRay") and cm == Method and self == camera then
                local origin = camera.CFrame.Position; return Ray.new(origin, (currentTargetPart.Position-origin).Unit)
            end
        end
        return oldNamecall(...)
    end))
    local oldRay = hookfunction(Ray.new, newcclosure(function(origin, dir)
        if Toggles.SilentEnabledToggle.Value and not checkcaller() and SilentAimSettings.SilentAimMethod == "Ray" and currentTargetPart and CalculateChance(SilentAimSettings.HitChance) then
            return oldRay(origin, getDirection(origin, currentTargetPart.Position))
        end
        return oldRay(origin, dir)
    end))
end

-- Silent Aim UI
local SilentTab = Tabs.SilentAim
local MainGroup = SilentTab:AddLeftGroupbox("主设置", "target")
MainGroup:AddToggle("SilentEnabledToggle", { Text = "启用", Default = false, Callback = function(v) SilentAimSettings.Enabled = v; if v then installSilentHooks() end end }):AddKeyPicker("SilentKeybind", { Default = SilentAimSettings.ToggleKey, SyncToggleState = true, Mode = "Toggle" })
MainGroup:AddToggle("TeamCheckToggle", { Text = "队伍检查", Default = false, Callback = function(v) SilentAimSettings.TeamCheck = v end })
MainGroup:AddToggle("VisibleCheckToggle", { Text = "可见性检查", Default = false, Callback = function(v) SilentAimSettings.VisibleCheck = v end })
MainGroup:AddToggle("WallbangToggleSA", { Text = "穿墙", Default = false, Callback = function(v) SilentAimSettings.Wallbang = v end })
MainGroup:AddToggle("LeakAndHitToggle", { Text = "漏打模式", Default = false, Callback = function(v) SilentAimSettings.LeakAndHitMode = v end })
MainGroup:AddDropdown("TargetModeDropdown", { Text = "目标种类", Values = {"玩家","NPC","所有"}, Default = SilentAimSettings.TargetMode, Callback = function(v) SilentAimSettings.TargetMode = v end })
MainGroup:AddDropdown("TargetPartDropdown", { Text = "目标部位", Values = {"Head","HumanoidRootPart","Random"}, Default = SilentAimSettings.TargetPart, Callback = function(v) SilentAimSettings.TargetPart = v end })
MainGroup:AddDropdown("MethodDropdown", { Text = "静默瞄准方式", Values = {"Raycast","FindPartOnRayWithIgnoreList","FindPartOnRayWithWhitelist","FindPartOnRay","ScreenPointToRay","ViewportPointToRay","Ray"}, Default = SilentAimSettings.SilentAimMethod, Callback = function(v) SilentAimSettings.SilentAimMethod = v end })
MainGroup:AddSlider("HitChanceSlider", { Text = "命中率", Default = 100, Min = 0, Max = 100, Rounding = 1, Suffix = "%", Callback = function(v) SilentAimSettings.HitChance = v end })
MainGroup:AddSlider("MaxDistanceSlider", { Text = "最大距离", Default = 500, Min = 10, Max = 2000, Rounding = 0, Suffix = "studs", Callback = function(v) SilentAimSettings.MaxDistance = v end })

local VisualsGroup = SilentTab:AddRightGroupbox("视觉效果")
VisualsGroup:AddToggle("FOVVisibleToggle", { Text = "显示FOV圈", Default = true, Callback = function(v) FOVCircleGui.Enabled = v; SilentAimSettings.FOVVisible = v end }):AddColorPicker("FOVColorPicker", { Default = Color3.fromRGB(54,57,241), Title = "FOV圈颜色" })
Options.FOVColorPicker:OnChanged(function(v) FOVStroke.Color = v end)
VisualsGroup:AddSlider("FOVRadiusSlider", { Text = "FOV圈半径", Min = 10, Max = 1000, Default = 130, Rounding = 0, Callback = function(v) FOVCircleFrame.Size = UDim2.fromOffset(v*2, v*2); SilentAimSettings.FOVRadius = v end })
VisualsGroup:AddToggle("FixedFOVToggle", { Text = "固定FOV (移动端)", Default = true, Callback = function(v) SilentAimSettings.FixedFOV = v end })
VisualsGroup:AddToggle("ShowTargetToggle", { Text = "显示目标", Default = false, Callback = function(v) SilentAimSettings.ShowSilentAimTarget = v end }):AddColorPicker("TargetIndicatorColorPicker", { Default = Color3.fromRGB(255,0,0), Title = "指示器颜色" })
Options.TargetIndicatorColorPicker:OnChanged(function(v) target_indicator_circle.Color = v; for _,l in ipairs(target_indicator_lines) do l.Color = v end end)
VisualsGroup:AddDropdown("IndicatorStyleDropdown", { Text = "指示器样式", Values = {"Circle","Triangle","Pentagram"}, Default = "Circle" })
VisualsGroup:AddSlider("TargetIndicatorRadiusSlider", { Text = "指示器大小", Min = 5, Max = 50, Default = 20, Rounding = 0, Callback = function(v) SilentAimSettings.TargetIndicatorRadius = v end })
VisualsGroup:AddToggle("IndicatorRotationToggle", { Text = "指示器旋转", Default = false, Callback = function(v) SilentAimSettings.IndicatorRotationEnabled = v end })
VisualsGroup:AddSlider("IndicatorRotationSpeedSlider", { Text = "旋转速度", Min = 0, Max = 10, Default = 1, Rounding = 1, Callback = function(v) SilentAimSettings.IndicatorRotationSpeed = v end })
VisualsGroup:AddToggle("IndicatorRainbowToggle", { Text = "启用彩虹色", Default = false, Callback = function(v) SilentAimSettings.IndicatorRainbowEnabled = v end })
VisualsGroup:AddSlider("IndicatorRainbowSpeedSlider", { Text = "颜色变换速度", Min = 0, Max = 10, Default = 1, Rounding = 1, Callback = function(v) SilentAimSettings.IndicatorRainbowSpeed = v end })
VisualsGroup:AddToggle("ShowTracerToggle", { Text = "显示追踪线", Default = false, Callback = function(v) SilentAimSettings.ShowTracer = v end }):AddColorPicker("TracerColorPicker", { Default = Color3.fromRGB(255,255,0), Title = "追踪线颜色" })
Options.TracerColorPicker:OnChanged(function(v) tracer_line.Color = v end)
VisualsGroup:AddSlider("TracerYOffsetSlider", { Text = "追踪线Y轴偏移", Default = 0, Min = -10, Max = 10, Rounding = 3, Suffix = " studs", Callback = function(v) SilentAimSettings.Tracer_Y_Offset = v end })

local PredictionGroup = SilentTab:AddLeftGroupbox("预判")
PredictionGroup:AddToggle("PredictionToggle", { Text = "Mouse.Hit/Target 预判", Default = false, Callback = function(v) SilentAimSettings.MouseHitPrediction = v end })
PredictionGroup:AddSlider("PredictionAmountSlider", { Text = "预判量", Default = 0.165, Min = 0, Max = 1, Rounding = 3, Callback = function(v) SilentAimSettings.MouseHitPredictionAmount = v; PredictionAmount = v end })

local MiscGroup = SilentTab:AddLeftGroupbox("杂项")
MiscGroup:AddDropdown("PriorityModeDropdown", { Text = "优先模式", Values = {"准星最近","距离最近","最低血量"}, Default = "准星最近", Callback = function(v) SilentAimSettings.PriorityMode = v end })
MiscGroup:AddDropdown("TargetInfoStyleDropdown", { Text = "信息显示样式", Values = {"面板","头顶","独立面板"}, Default = "面板", Callback = function(v) SilentAimSettings.TargetInfoStyle = v end })
MiscGroup:AddToggle("ShowTargetNameToggle", { Text = "显示目标名字", Default = false, Callback = function(v) SilentAimSettings.ShowTargetName = v end })
MiscGroup:AddToggle("ShowTargetHealthToggle", { Text = "显示目标血量", Default = false, Callback = function(v) SilentAimSettings.ShowTargetHealth = v end })
MiscGroup:AddToggle("ShowTargetDistanceToggle", { Text = "显示目标距离", Default = false, Callback = function(v) SilentAimSettings.ShowTargetDistance = v end })
MiscGroup:AddToggle("ShowTargetCategoryToggle", { Text = "显示目标类别", Default = false, Callback = function(v) SilentAimSettings.ShowTargetCategory = v end })
MiscGroup:AddToggle("DamageNotifierToggle", { Text = "显示伤害通知", Default = false, Callback = function(v) SilentAimSettings.ShowDamageNotifier = v end })
MiscGroup:AddButton("重置独立面板位置", function()
    SilentAimSettings.IndependentPanelPosition = "200,200"
    local pos = SilentAimSettings.IndependentPanelPosition:split(",")
    IndependentPanelFrame.Position = UDim2.fromOffset(tonumber(pos[1]), tonumber(pos[2]))
end)
MiscGroup:AddToggle("PinPanelToggle", { Text = "固定面板", Default = false, Callback = function(v) SilentAimSettings.IndependentPanelPinned = v; IndependentPanelFrame.Draggable = not v end })

local LockGroup = SilentTab:AddLeftGroupbox("锁定")
LockGroup:AddDropdown("TargetSelectorDropdown", { Text = "锁定目标", Default = "无", Values = {"无"}, Callback = function(v) lockedTargetCharacter = v~="无" and targetMap[v] or nil end })
LockGroup:AddButton("刷新列表", function()
    targetMap = {}; local names = {"无"}
    if SilentAimSettings.TargetMode=="NPC" or SilentAimSettings.TargetMode=="所有" then updateNPCs() end
    if SilentAimSettings.TargetMode=="玩家" or SilentAimSettings.TargetMode=="所有" then
        for _, p in ipairs(Players:GetPlayers()) do
            if p~=player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                if not (SilentAimSettings.TeamCheck and not IsEnemy(p.Character)) then
                    table.insert(names, p.Name); targetMap[p.Name]=p.Character
                end
            end
        end
    end
    if SilentAimSettings.TargetMode=="NPC" or SilentAimSettings.TargetMode=="所有" then
        for _, npc in ipairs(npcList) do
            if npc and npc.Name and npc.PrimaryPart then table.insert(names, npc.Name); targetMap[npc.Name]=npc end
        end
    end
    Options.TargetSelectorDropdown:SetValues(names, "无")
    lockedTargetCharacter = nil
end)

local NameGroup = SilentTab:AddRightGroupbox("名称索敌")
NameGroup:AddToggle("EnableNameTargetingToggle", { Text = "启用名称索敌", Default = false, Callback = function(v) SilentAimSettings.EnableNameTargeting = v end })
NameGroup:AddInput("TargetName1Input", { Text = "目标名称 1", Default = "", PlaceholderText = "输入NPC名称关键字", Callback = function(v) SilentAimSettings.TargetName1 = v end })
NameGroup:AddInput("TargetName2Input", { Text = "目标名称 2", Default = "", PlaceholderText = "输入NPC名称关键字", Callback = function(v) SilentAimSettings.TargetName2 = v end })
NameGroup:AddInput("TargetName3Input", { Text = "目标名称 3", Default = "", PlaceholderText = "输入NPC名称关键字", Callback = function(v) SilentAimSettings.TargetName3 = v end })

FOVCircleGui.Enabled = Toggles.FOVVisibleToggle.Value
FOVCircleFrame.Size = UDim2.fromOffset(260, 260)
IndependentPanelFrame.Draggable = not SilentAimSettings.IndependentPanelPinned
task.spawn(function() while task.wait(2) do if SilentAimSettings.TargetMode=="NPC" or SilentAimSettings.TargetMode=="所有" then updateNPCs() end end end)

-- ==================== SKINS ====================
local scriptRunning = false
local selectedKnife = "Butterfly Knife"
local spawned = false; local inspecting = false; local swinging = false; local lastAttackTime = 0
local ATTACK_COOLDOWN = 1
local ACTION_INSPECT = "InspectKnifeAction"; local ACTION_ATTACK = "AttackKnifeAction"
pcall(function() RS.Assets.Weapons.Karambit.Camera.ViewmodelLight.Transparency = 1 end)
local knives = {
    ["Karambit"]={Offset=CFrame.new(0,-1.5,1.5)}, ["Butterfly Knife"]={Offset=CFrame.new(0,-1.5,1.5)},
    ["M9 Bayonet"]={Offset=CFrame.new(0,-1.5,1)}, ["Flip Knife"]={Offset=CFrame.new(0,-1.5,1.25)},
    ["Gut Knife"]={Offset=CFrame.new(0,-1.5,0.5)}, ["Stiletto Knife"]={Offset=CFrame.new(0,-1.5,1.25)},
    ["Skeleton Knife"]={Offset=CFrame.new(0,-1.5,1.25)}
}
local vm, animator
local equipAnim, idleAnim, inspectAnim, HeavySwingAnim, Swing1Anim, Swing2Anim
local function getKnifeInCamera() return camera:FindFirstChild("T Knife") or camera:FindFirstChild("CT Knife") end
local function cleanPart(p) if p:IsA("BasePart") then p.CanCollide,p.Anchored,p.CastShadow,p.CanTouch,p.CanQuery = false,false,false,false,false end end
local function disableCollisions(m) for _,p in m:GetDescendants() do cleanPart(p) end end
local function hideOriginalKnife(k) for _,p in k:GetDescendants() do if p:IsA("BasePart") or p:IsA("MeshPart") or p:IsA("Texture") then p.Transparency=1 end end end
local function playSound(folder,name)
    local ws = RS.Sounds:FindFirstChild(selectedKnife); if not ws then return end
    local s = ws:WaitForChild(folder):WaitForChild(name):Clone(); s.Parent=camera; s:Play(); s.Ended:Once(function() s:Destroy() end); return s
end
local function attachAsset(folder,armPart,assetModel,finalName,offset)
    local arm = vm:FindFirstChild(armPart); if not arm then return end
    local mesh = folder:WaitForChild(assetModel):Clone(); cleanPart(mesh); mesh.Name=finalName; mesh.Parent=arm
    local motor = Instance.new("Motor6D"); motor.Part0,motor.Part1,motor.C0,motor.Parent = arm,mesh,offset,arm
end
local function handleAction(actionName, inputState, inputObject)
    if inputState ~= Enum.UserInputState.Begin or not spawned or not animator or not isAlive() then return Enum.ContextActionResult.Pass end
    if actionName == ACTION_INSPECT then
        if (equipAnim and equipAnim.IsPlaying) or inspecting or swinging then return Enum.ContextActionResult.Pass end
        inspecting = true; if idleAnim then idleAnim:Stop() end; inspectAnim:Play(); inspectAnim.Stopped:Once(function() inspecting=false end)
    elseif actionName == ACTION_ATTACK then
        local ct = os.clock()
        if (equipAnim and equipAnim.IsPlaying) or (ct-lastAttackTime<ATTACK_COOLDOWN) then return Enum.ContextActionResult.Pass end
        lastAttackTime = ct; if inspecting then inspecting=false; if inspectAnim then inspectAnim:Stop() end end
        swinging=true; if idleAnim then idleAnim:Stop() end
        local anims = {HeavySwingAnim,Swing1Anim,Swing2Anim}; local chosen = anims[math.random(1,#anims)]
        local sf = (chosen==HeavySwingAnim and "HitOne") or (chosen==Swing1Anim and "HitTwo") or "HitThree"
        chosen:Play(); local s = playSound(sf,"1"); if s then s.Volume=5 end; chosen.Stopped:Once(function() swinging=false end)
    end
    return Enum.ContextActionResult.Pass
end
local function removeViewmodel()
    spawned=false; CAS:UnbindAction(ACTION_INSPECT); CAS:UnbindAction(ACTION_ATTACK)
    if vm then vm:Destroy() vm=nil end; animator,inspecting,swinging = nil,false,false
end
local function spawnViewmodel(knife)
    if spawned or not scriptRunning then return end
    local myModel = isAlive(); if not myModel then return end
    spawned=true; local knifeTemplate = RS.Assets.Weapons:WaitForChild(selectedKnife)
    local knifeOffset = knives[selectedKnife].Offset
    vm = knifeTemplate:WaitForChild("Camera"):Clone(); vm.Name,vm.Parent = selectedKnife,camera
    disableCollisions(vm); hideOriginalKnife(knife)
    if myModel.Parent.Name == "Terrorists" then
        local tg = RS.Assets.Weapons:WaitForChild("T Glove")
        attachAsset(tg,"Left Arm","Left Arm","Glove",CFrame.new(0,0,-1.5))
        attachAsset(tg,"Right Arm","Right Arm","Glove",CFrame.new(0,0,-1.5))
    else
        local sleeves = RS.Assets.Sleeves:WaitForChild("IDF"); local ctg = RS.Assets.Weapons:WaitForChild("CT Glove")
        attachAsset(sleeves,"Left Arm","Left Arm","Sleeve",CFrame.new(0,0,0.5))
        attachAsset(ctg,"Left Arm","Left Arm","Glove",CFrame.new(0,0,-1.5))
        attachAsset(sleeves,"Right Arm","Right Arm","Sleeve",CFrame.new(0,0,0.5))
        attachAsset(ctg,"Right Arm","Right Arm","Glove",CFrame.new(0,0,-1.5))
    end
    local ac = vm:FindFirstChildOfClass("AnimationController") or vm:FindFirstChildOfClass("Animator")
    animator = ac:FindFirstChildWhichIsA("Animator") or ac
    local af = RS.Assets.WeaponAnimations:WaitForChild(selectedKnife):WaitForChild("CameraAnimations")
    equipAnim = animator:LoadAnimation(af:WaitForChild("Equip"))
    idleAnim = animator:LoadAnimation(af:WaitForChild("Idle"))
    inspectAnim = animator:LoadAnimation(af:WaitForChild("Inspect"))
    HeavySwingAnim = animator:LoadAnimation(af:WaitForChild("Heavy Swing"))
    Swing1Anim = animator:LoadAnimation(af:WaitForChild("Swing1"))
    Swing2Anim = animator:LoadAnimation(af:WaitForChild("Swing2"))
    vm:SetPrimaryPartCFrame(camera.CFrame * CFrame.new(0,-1.5,5))
    TweenService:Create(vm.PrimaryPart, TweenInfo.new(0.2,Enum.EasingStyle.Quad,Enum.EasingDirection.Out), {CFrame=camera.CFrame*knifeOffset}):Play()
    equipAnim:Play(); playSound("Equip","1")
    CAS:BindAction(ACTION_INSPECT,handleAction,false,Enum.KeyCode.F)
    CAS:BindAction(ACTION_ATTACK,handleAction,false,Enum.UserInputType.MouseButton1)
end
RunService.RenderStepped:Connect(function()
    if not scriptRunning or not vm or not vm.PrimaryPart then return end
    vm.PrimaryPart.CFrame = camera.CFrame * knives[selectedKnife].Offset
    if not (equipAnim and equipAnim.IsPlaying) and not inspecting and not swinging then
        if idleAnim and not idleAnim.IsPlaying then idleAnim:Play() end
    end
end)
task.spawn(function()
    while task.wait(0.1) do
        local living = isAlive(); local currentKnife = getKnifeInCamera()
        if scriptRunning and living and currentKnife and not spawned then spawnViewmodel(currentKnife)
        elseif (not scriptRunning or not currentKnife or not living) and spawned then removeViewmodel() end
    end
end)

-- Skin Changer
local SkinChangerEnabled = false
local SelectedSkins = {}; local DropdownObjects = {}; local SkinOptions = {}; local COOLDOWN = 0.1; local WEAR = "Factory New"
local CT_ONLY = {["USP-S"]=true,["Five-SeveN"]=true,["MP9"]=true,["FAMAS"]=true,["M4A1-S"]=true,["M4A4"]=true,["AUG"]=true}
local SHARED = {["P250"]=true,["Desert Eagle"]=true,["Dual Berettas"]=true,["Negev"]=true,["P90"]=true,["Nova"]=true,["XM1014"]=true,["AWP"]=true,["SSG 08"]=true}
local KNIVES = {["Karambit"]=true,["Butterfly Knife"]=true,["M9 Bayonet"]=true,["Flip Knife"]=true,["Gut Knife"]=true,["T Knife"]=true,["CT Knife"]=true,["Stiletto Knife"]=true,["Skeleton Knife"]=true}
local GLOVES = {["Sports Gloves"]=true}
local SkinsFolder = RS:WaitForChild("Assets"):WaitForChild("Skins")
local IgnoreFolders = {["HE Grenade"]=true,["Incendiary Grenade"]=true,["Molotov"]=true,["Smoke Grenade"]=true,["Flashbang"]=true,["Decoy Grenade"]=true,["C4"]=true,["CT Glove"]=true,["T Glove"]=true}
local function getAllSkins(f) local s={}; for _,sk in f:GetChildren() do table.insert(s,sk.Name) end; return s end
local function applyWeaponSkin(model)
    if not model or not SkinChangerEnabled or not isAlive() then return end
    local skinName = SelectedSkins[model.Name]; if not skinName then return end
    pcall(function()
        local skinFolder = SkinsFolder:FindFirstChild(model.Name); if not skinFolder then return end
        local skinType = skinFolder:FindFirstChild(skinName)
        local sourceFolder = skinType and skinType:FindFirstChild("Camera") and skinType.Camera:FindFirstChild(WEAR); if not sourceFolder then return end
        for _, obj in camera:GetChildren() do
            local left,right = obj:FindFirstChild("Left Arm"),obj:FindFirstChild("Right Arm")
            if left or right then
                local gf = SkinsFolder:FindFirstChild("Sports Gloves"); local gs = gf and gf:FindFirstChild(SelectedSkins["Sports Gloves"])
                local gsrc = gs and gs:FindFirstChild("Camera") and gs.Camera:FindFirstChild(WEAR)
                if gsrc then
                    for _, side in ipairs({"Left Arm","Right Arm"}) do
                        local arm,src = obj:FindFirstChild(side),gsrc:FindFirstChild(side)
                        if arm and src then
                            local gloveMesh = arm:FindFirstChild("Glove")
                            if gloveMesh then
                                local ex = gloveMesh:FindFirstChildOfClass("SurfaceAppearance"); if ex then ex:Destroy() end
                                local c = src:Clone(); c.Name,c.Parent = "SurfaceAppearance",gloveMesh
                            end
                        end
                    end
                end
            end
        end
        if not GLOVES[model.Name] then
            local wf = model:FindFirstChild("Weapon")
            if wf then
                for _, part in wf:GetDescendants() do
                    if part:IsA("BasePart") then
                        local ns = sourceFolder:FindFirstChild(part.Name)
                        if ns then
                            local ex = part:FindFirstChildOfClass("SurfaceAppearance"); if ex then ex:Destroy() end
                            local c = ns:Clone(); c.Name,c.Parent = "SurfaceAppearance",part
                        end
                    end
                end
            end
        end
        model:SetAttribute("SkinApplied",skinName)
    end)
end

local SkinsGroup = Tabs.Skins:AddLeftGroupbox("皮肤修改器", "palette")
SkinsGroup:AddToggle("SkinChangerToggle", { Text = "启用皮肤修改器", Default = false, Callback = function(v) SkinChangerEnabled=v; if not v then for _,obj in camera:GetChildren() do obj:SetAttribute("SkinApplied",nil) end end end })
SkinsGroup:AddButton({ Text = "随机所有皮肤", Func = function() for wn,ol in pairs(SkinOptions) do if #ol>0 then local rs=ol[math.random(1,#ol)]; if DropdownObjects[wn] then for _,dd in ipairs(DropdownObjects[wn]) do dd:SetValue(rs) end end end end end })
local KnifeGroup = Tabs.Skins:AddRightGroupbox("自定义刀子", "swords")
KnifeGroup:AddToggle("KnifeToggle", { Text = "启用自定义刀子", Default = false, Callback = function(v) scriptRunning=v; if not v then removeViewmodel() end end })
KnifeGroup:AddDropdown("KnifeDropdown", { Text = "选择自定义刀子", Values = {"Butterfly Knife","Karambit","M9 Bayonet","Flip Knife","Gut Knife","Stiletto Knife","Skeleton Knife"}, Default = "Butterfly Knife", Callback = function(v) selectedKnife=v; if spawned then removeViewmodel() end end })
local SkinsRightGroup = Tabs.Skins:AddRightGroupbox("武器皮肤", "palette")
local function CreateSkinDropdown(weaponName, group)
    local folder = SkinsFolder:FindFirstChild(weaponName); if not folder then return end
    local options = getAllSkins(folder); SkinOptions[weaponName] = options
    if #options>0 then if not SelectedSkins[weaponName] then SelectedSkins[weaponName]=options[1] end else SelectedSkins[weaponName]=nil end
    local dp = group:AddDropdown("Skin_"..weaponName:gsub("%W",""), {
        Name = weaponName, Text = weaponName, Values = options, Default = SelectedSkins[weaponName] or (options[1] or ""),
        Callback = function(opt) SelectedSkins[weaponName]=opt;
            if DropdownObjects[weaponName] then for _,other in ipairs(DropdownObjects[weaponName]) do if other.Value~=opt then other:SetValue(opt) end end end
            for _,obj in camera:GetChildren() do obj:SetAttribute("SkinApplied",nil); applyWeaponSkin(obj) end
        end
    })
    DropdownObjects[weaponName] = DropdownObjects[weaponName] or {}; table.insert(DropdownObjects[weaponName],dp)
end
SkinsRightGroup:AddDivider(); SkinsRightGroup:AddLabel("刀具皮肤"); for name in pairs(KNIVES) do CreateSkinDropdown(name,SkinsRightGroup) end
SkinsRightGroup:AddDivider(); SkinsRightGroup:AddLabel("手套"); for name in pairs(GLOVES) do CreateSkinDropdown(name,SkinsRightGroup) end
SkinsRightGroup:AddDivider(); SkinsRightGroup:AddLabel("CT武器"); for name in pairs(CT_ONLY) do CreateSkinDropdown(name,SkinsRightGroup) end
SkinsRightGroup:AddDivider(); SkinsRightGroup:AddLabel("T武器"); for name in pairs(SHARED) do CreateSkinDropdown(name,SkinsRightGroup) end
for _, folder in SkinsFolder:GetChildren() do
    local n = folder.Name
    if not IgnoreFolders[n] and not KNIVES[n] and not GLOVES[n] and not CT_ONLY[n] and not SHARED[n] then CreateSkinDropdown(n,SkinsRightGroup) end
end
camera.ChildAdded:Connect(function(obj) if not SkinChangerEnabled or not isAlive() then return end; task.wait(COOLDOWN); applyWeaponSkin(obj) end)
task.spawn(function() while task.wait(0.5) do if SkinChangerEnabled and isAlive() then for _,obj in camera:GetChildren() do if SelectedSkins[obj.Name] and obj:GetAttribute("SkinApplied")~=SelectedSkins[obj.Name] then applyWeaponSkin(obj) end end end end end)

-- ==================== VISUALS (ESP) ====================
local Esp = {Enabled = false, Box = true, Name = true, Health = true, Distance = true, Skeleton = true}
local espCache = {}
local function createESP()
    local esp = {boxOutline=Drawing.new("Square"),box=Drawing.new("Square"),name=Drawing.new("Text"),distance=Drawing.new("Text"),healthOutline=Drawing.new("Line"),healthBar=Drawing.new("Line"),bones={}}
    esp.boxOutline.Thickness=3; esp.boxOutline.Filled=false; esp.boxOutline.Color=Color3.new(0,0,0)
    esp.box.Thickness=1; esp.box.Filled=false; esp.box.Color=Color3.fromRGB(255,50,50)
    esp.name.Center=true; esp.name.Outline=true; esp.name.Color=Color3.new(1,1,1); esp.name.Size=16
    esp.distance.Center=true; esp.distance.Outline=true; esp.distance.Color=Color3.new(0.8,0.8,0.8); esp.distance.Size=13
    esp.healthOutline.Thickness=3; esp.healthOutline.Color=Color3.new(0,0,0)
    esp.healthBar.Thickness=2; esp.healthBar.Color=Color3.new(0,1,0)
    local bonePairs={"Head_UpperTorso","UpperTorso_LowerTorso","UpperTorso_LeftUpperArm","LeftUpperArm_LeftLowerArm","LeftLowerArm_LeftHand","UpperTorso_RightUpperArm","RightUpperArm_RightLowerArm","RightLowerArm_RightHand","LowerTorso_LeftUpperLeg","LeftUpperLeg_LeftLowerLeg","LeftLowerLeg_LeftFoot","LowerTorso_RightUpperLeg","RightUpperLeg_RightLowerLeg","RightLowerLeg_RightFoot"}
    for _,name in ipairs(bonePairs) do local line=Drawing.new("Line") line.Thickness=1.5 line.Color=Color3.fromRGB(255,255,255) line.Transparency=0.8 line.Visible=false esp.bones[name]=line end
    return esp
end
local function w2s(pos) local sp,on=camera:WorldToViewportPoint(pos) return Vector2.new(sp.X,sp.Y),on end
local function updateSkeleton(esp,model)
    local function getPart(n) return model:FindFirstChild(n) end
    local connections={{"Head","UpperTorso"},{"UpperTorso","LowerTorso"},{"UpperTorso","LeftUpperArm"},{"LeftUpperArm","LeftLowerArm"},{"LeftLowerArm","LeftHand"},{"UpperTorso","RightUpperArm"},{"RightUpperArm","RightLowerArm"},{"RightLowerArm","RightHand"},{"LowerTorso","LeftUpperLeg"},{"LeftUpperLeg","LeftLowerLeg"},{"LeftLowerLeg","LeftFoot"},{"LowerTorso","RightUpperLeg"},{"RightUpperLeg","RightLowerLeg"},{"RightLowerLeg","RightFoot"}}
    for _,pair in ipairs(connections) do local p1,p2=getPart(pair[1]),getPart(pair[2]) local pos1,on1=p1 and w2s(p1.Position) local pos2,on2=p2 and w2s(p2.Position) local line=esp.bones[pair[1].."_"..pair[2]] if line then if on1 and on2 then line.From=pos1 line.To=pos2 line.Visible=true else line.Visible=false end end end
end
RunService.RenderStepped:Connect(function()
    if not Esp.Enabled or not isAlive() then
        for _,esp in pairs(espCache) do
            esp.boxOutline.Visible=false; esp.box.Visible=false; esp.name.Visible=false; esp.distance.Visible=false
            esp.healthOutline.Visible=false; esp.healthBar.Visible=false
            for _,l in pairs(esp.bones) do l.Visible=false end
        end
        return
    end
    local ef = getEnemyFolder(); if not ef then return end
    local curAlive = {}
    for _,enemy in ipairs(ef:GetChildren()) do
        local hum,root,head = enemy:FindFirstChildOfClass("Humanoid"), enemy:FindFirstChild("HumanoidRootPart"), enemy:FindFirstChild("Head")
        if hum and hum.Health>0 and root and head then
            curAlive[enemy]=true
            if not espCache[enemy] then espCache[enemy]=createESP() end
            local esp = espCache[enemy]
            local rootPos,rootOn = camera:WorldToViewportPoint(root.Position)
            local headScr = camera:WorldToViewportPoint(head.Position+Vector3.new(0,0.5,0))
            local legScr = camera:WorldToViewportPoint(root.Position-Vector3.new(0,3,0))
            if rootOn then
                local boxH = math.abs(headScr.Y-legScr.Y); local boxW = boxH*0.55
                local boxX = rootPos.X-boxW/2; local boxY = headScr.Y
                local dist = math.floor((camera.CFrame.Position-root.Position).Magnitude)
                if Esp.Box then esp.boxOutline.Size=Vector2.new(boxW,boxH); esp.boxOutline.Position=Vector2.new(boxX,boxY); esp.boxOutline.Visible=true esp.box.Size=Vector2.new(boxW,boxH); esp.box.Position=Vector2.new(boxX,boxY); esp.box.Visible=true else esp.boxOutline.Visible=false; esp.box.Visible=false end
                if Esp.Health then local hpPct=hum.Health/hum.MaxHealth; local barX=boxX-7 esp.healthOutline.From=Vector2.new(barX,boxY-1); esp.healthOutline.To=Vector2.new(barX,boxY+boxH+1); esp.healthOutline.Visible=true esp.healthBar.From=Vector2.new(barX,boxY+boxH); esp.healthBar.To=Vector2.new(barX,boxY+boxH-(boxH*hpPct)); esp.healthBar.Color=Color3.new(1-hpPct,hpPct,0); esp.healthBar.Visible=true else esp.healthOutline.Visible=false; esp.healthBar.Visible=false end
                if Esp.Name then esp.name.Text=enemy.Name; esp.name.Position=Vector2.new(rootPos.X,boxY-22); esp.name.Visible=true else esp.name.Visible=false end
                if Esp.Distance then esp.distance.Text="["..dist.."m]"; esp.distance.Position=Vector2.new(rootPos.X,boxY+boxH+4); esp.distance.Visible=true else esp.distance.Visible=false end
                if Esp.Skeleton then updateSkeleton(esp,enemy) else for _,l in pairs(esp.bones) do l.Visible=false end end
            else
                esp.boxOutline.Visible=false; esp.box.Visible=false; esp.name.Visible=false; esp.distance.Visible=false; esp.healthOutline.Visible=false; esp.healthBar.Visible=false for _,l in pairs(esp.bones) do l.Visible=false end
            end
        end
    end
    for enemy,esp in pairs(espCache) do
        if not curAlive[enemy] then
            esp.boxOutline:Remove(); esp.box:Remove(); esp.name:Remove(); esp.distance:Remove(); esp.healthOutline:Remove(); esp.healthBar:Remove()
            for _,l in pairs(esp.bones) do l:Remove() end
            espCache[enemy]=nil
        end
    end
end)
local EspGroup = Tabs.Visuals:AddLeftGroupbox("ESP", "eye")
EspGroup:AddToggle("ESPToggle", { Text = "启用玩家 ESP", Default = false, Callback = function(v) Esp.Enabled=v end })
local EspSettings = Tabs.Visuals:AddLeftGroupbox("ESP 设置", "eye")
EspSettings:AddToggle("EspBoxToggle", { Text = "方框", Default = true, Callback = function(v) Esp.Box=v end })
EspSettings:AddToggle("EspHealthToggle", { Text = "血量", Default = true, Callback = function(v) Esp.Health=v end })
EspSettings:AddToggle("EspNameToggle", { Text = "名称", Default = true, Callback = function(v) Esp.Name=v end })
EspSettings:AddToggle("EspDistanceToggle", { Text = "距离", Default = true, Callback = function(v) Esp.Distance=v end })
EspSettings:AddToggle("EspSkeletonToggle", { Text = "骨骼", Default = true, Callback = function(v) Esp.Skeleton=v end })

-- World Effects
local AntiFlashEnabled, AntiSmokeEnabled = false, false
local WorldGroup = Tabs.Visuals:AddRightGroupbox("世界效果", "sun")
WorldGroup:AddToggle("AntiFlashToggle", { Text = "防闪光弹", Default = false, Callback = function(v) AntiFlashEnabled=v end })
WorldGroup:AddToggle("AntiSmokeToggle", { Text = "防烟雾弹", Default = false, Callback = function(v) AntiSmokeEnabled=v end })
task.spawn(function() while task.wait(0.2) do if AntiFlashEnabled then local g = player.PlayerGui:FindFirstChild("FlashbangEffect"); if g then g:Destroy() end; local e = game:GetService("Lighting"):FindFirstChild("FlashbangColorCorrection"); if e then e:Destroy() end end end end)
task.spawn(function() while task.wait(0.5) do if AntiSmokeEnabled then local d = Workspace:FindFirstChild("Debris"); if d then for _,f in ipairs(d:GetChildren()) do if string.match(f.Name,"Voxel") then f:ClearAllChildren(); f:Destroy() end end end end end end)

-- ==================== UI SETTINGS ====================
local MenuGroup = Tabs["UI Settings"]:AddLeftGroupbox("菜单", "wrench")
MenuGroup:AddToggle("KeybindMenuOpen", { Default = false, Text = "打开快捷键菜单", Callback = function(v) Library.KeybindFrame.Visible=v end })
MenuGroup:AddToggle("ShowCustomCursor", { Text = "自定义光标", Default = true, Callback = function(v) Library.ShowCustomCursor=v end })
MenuGroup:AddDropdown("NotificationSide", { Values = {"左","右"}, Default = "右", Text = "通知位置", Callback = function(v) Library:SetNotifySide(v) end })
MenuGroup:AddDropdown("DPIDropdown", { Values = {"50%","75%","100%","125%","150%","175%","200%"}, Default = "100%", Text = "DPI缩放", Callback = function(v) v=v:gsub("%%",""); Library:SetDPIScale(tonumber(v)) end })
MenuGroup:AddDivider()
MenuGroup:AddLabel("菜单热键"):AddKeyPicker("MenuKeybind", { Default = "RightShift", NoUI = true, Text = "菜单热键" })
MenuGroup:AddButton("卸载脚本", function() Library:Unload() end)
Library.ToggleKeybind = Options.MenuKeybind

-- 主题与保存（带保护，延迟初始化）
task.spawn(function()
    task.wait(0.5)
    pcall(function()
        ThemeManager:SetLibrary(Library)
        SaveManager:SetLibrary(Library)
        SaveManager:IgnoreThemeSettings()
        SaveManager:SetIgnoreIndexes({ "MenuKeybind" })
        ThemeManager:SetFolder(winTitle)
        SaveManager:SetFolder(winTitle)
        SaveManager:SetSubFolder("BloxStrike")
        SaveManager:BuildConfigSection(Tabs["UI Settings"])
        ThemeManager:ApplyToTab(Tabs["UI Settings"])
        SaveManager:LoadAutoloadConfig()
    end)
end)

Library:OnUnload(function()
    pcall(function() FOVCircleGui:Destroy() end)
    pcall(function() if IndependentPanelGui then IndependentPanelGui:Destroy() end end)
    hideAllVisuals()
    print("已卸载")
end)