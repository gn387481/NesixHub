--[[
    Nesix Hub - Professional Script
    Powered by LinoriaLib
]]

local repo = "https://raw.githubusercontent.com/mstudio45/LinoriaLib/main/"

local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()
local ThemeManager = loadstring(game:HttpGet(repo .. "addons/ThemeManager.lua"))()
local SaveManager = loadstring(game:HttpGet(repo .. "addons/SaveManager.lua"))()

local Options = Library.Options
local Toggles = Library.Toggles

Library.ShowToggleFrameInKeybinds = true
Library.ShowCustomCursor = true
Library.NotifySide = "Left"

local Window = Library:CreateWindow({
    Title = "                          Nesix Hub                          ",
    Center = true,
    AutoShow = true,
    Resizable = true,
    ShowCustomCursor = true,
    UnlockMouseWhileOpen = true,
    NotifySide = "Left",
    TabPadding = 8,
    MenuFadeTime = 0.2
})

local Tabs = {
    Main = Window:AddTab("Main"),
    ESP = Window:AddTab("ESP"),
    Visual = Window:AddTab("Visual"),
    Misc = Window:AddTab("Misc"),
    ["UI Settings"] = Window:AddTab("UI Settings"),
}

-- =============================================
--                  SILENT AIM
-- =============================================
local SilentAimEnabled = false
local SilentRageEnabled = false
local ShowFOVCircle = true
local MaxDistance = 500
local FOVRadius = 150
local AimTargetPart = "Head"

local Camera = workspace.CurrentCamera
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")
local TextChatService = game:GetService("TextChatService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local fovCircle = Drawing.new("Circle")
fovCircle.Thickness = 1.5
fovCircle.Filled = false
fovCircle.NumSides = 64
fovCircle.Visible = false
fovCircle.Color = Color3.fromRGB(255, 50, 50)

local function GetClosestTarget(origin)
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local bestTarget, bestDist = nil, math.huge
    local myChar = LocalPlayer.Character

    for _, player in pairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local char = player.Character
        if not char or char == myChar then continue end
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then continue end

        local targetPart = char:FindFirstChild(AimTargetPart) or char:FindFirstChild("Head")
        if not targetPart then continue end
        if (origin - targetPart.Position).Magnitude > MaxDistance then continue end

        if SilentRageEnabled then
            local dist = (origin - targetPart.Position).Magnitude
            if dist < bestDist then
                bestDist = dist
                bestTarget = targetPart
            end
        else
            local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
            if not onScreen then continue end
            local dist = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
            if dist < FOVRadius and dist < bestDist then
                bestDist = dist
                bestTarget = targetPart
            end
        end
    end
    return bestTarget
end

RunService.RenderStepped:Connect(function()
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    fovCircle.Position = screenCenter
    fovCircle.Radius = FOVRadius
    fovCircle.Visible = SilentAimEnabled and ShowFOVCircle and not SilentRageEnabled

    local myRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    local origin = myRoot and myRoot.Position or Camera.CFrame.Position
    fovCircle.Color = GetClosestTarget(origin) and Color3.fromRGB(255,50,50) or Color3.fromRGB(0,200,255)
end)

local success, util = pcall(function() return require(game:GetService("ReplicatedStorage").Modules.Utility) end)
if success and util then
    local originalRaycast = util.Raycast
    util.Raycast = function(self, origin, direction, distance, ...)
        if SilentAimEnabled then
            local target = GetClosestTarget(origin)
            if target then
                return originalRaycast(self, origin, target.Position, distance, ...)
            end
        end
        return originalRaycast(self, origin, direction, distance, ...)
    end
end

local SilentGroup = Tabs.Main:AddLeftGroupbox("Silent Aim")
SilentGroup:AddToggle("SilentAimToggle", { Text = "Silent Aim", Default = false, Callback = function(v) SilentAimEnabled = v end })
SilentGroup:AddToggle("SilentRageToggle", { Text = "Silent Rage (Ignore FOV)", Default = false, Callback = function(v) SilentRageEnabled = v end })
SilentGroup:AddToggle("ShowFOVToggle", { Text = "Show FOV Circle", Default = true, Callback = function(v) ShowFOVCircle = v end })
SilentGroup:AddSlider("FOVSlider", { Text = "FOV Radius", Default = 150, Min = 50, Max = 500, Rounding = 0, Callback = function(v) FOVRadius = v end })
SilentGroup:AddDropdown("TargetPartDropdown", { Text = "Target Part", Values = {"Head", "UpperTorso", "HumanoidRootPart"}, Default = 1, Callback = function(v) AimTargetPart = v end })

-- =============================================
--                  ESP
-- =============================================
local ESPBoxes = false
local ESPNames = false
local ESPHealth = false
local ESPTracers = false

local ESPObjects = {}

local function CreateESP(plr)
    if ESPObjects[plr] then return end
    local o = {}
    o.box = Drawing.new("Square")
    o.box.Thickness = 1
    o.box.Filled = false
    o.box.Color = Color3.fromRGB(255,255,255)

    o.outline = Drawing.new("Square")
    o.outline.Thickness = 3
    o.outline.Filled = false
    o.outline.Color = Color3.new(0,0,0)
    o.outline.Transparency = 0.6

    o.name = Drawing.new("Text")
    o.name.Size = 13
    o.name.Center = true
    o.name.Outline = true
    o.name.Color = Color3.fromRGB(255,255,255)
    o.name.Font = 2

    o.tracer = Drawing.new("Line")
    o.tracer.Thickness = 1
    o.tracer.Color = Color3.fromRGB(120,170,255)

    o.healthBarBG = Drawing.new("Square")
    o.healthBarBG.Thickness = 1
    o.healthBarBG.Filled = true
    o.healthBarBG.Color = Color3.new(0,0,0)

    o.healthBar = Drawing.new("Square")
    o.healthBar.Thickness = 1
    o.healthBar.Filled = true
    o.healthBar.Color = Color3.fromRGB(80,220,120)

    ESPObjects[plr] = o
end

local function UpdateESP()
    for _, plr in pairs(Players:GetPlayers()) do
        if plr == LocalPlayer then continue end
        local char = plr.Character
        local data = ESPObjects[plr]
        if not char then
            if data then
                for _, d in pairs(data) do d.Visible = false end
            end
            continue
        end

        local root = char:FindFirstChild("HumanoidRootPart")
        local head = char:FindFirstChild("Head")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not data then CreateESP(plr) data = ESPObjects[plr] end
        if not root or not head or not hum or hum.Health <= 0 then
            for _, d in pairs(data) do d.Visible = false end
            continue
        end

        local headPos, onScreen = Camera:WorldToViewportPoint(head.Position + Vector3.new(0,0.6,0))
        local bottomPos = Camera:WorldToViewportPoint(root.Position - Vector3.new(0,3.2,0))
        if not onScreen then
            for _, d in pairs(data) do d.Visible = false end
            continue
        end

        local height = math.abs(bottomPos.Y - headPos.Y)
        local width = height * 0.5
        local x = headPos.X - width/2
        local y = headPos.Y

        data.box.Visible = ESPBoxes
        data.box.Size = Vector2.new(width, height)
        data.box.Position = Vector2.new(x, y)

        data.outline.Visible = ESPBoxes
        data.outline.Size = Vector2.new(width, height)
        data.outline.Position = Vector2.new(x, y)

        data.name.Visible = ESPNames
        data.name.Text = plr.DisplayName
        data.name.Position = Vector2.new(headPos.X, y - 16)

        data.tracer.Visible = ESPTracers
        data.tracer.From = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
        data.tracer.To = Vector2.new(headPos.X, y + height)

        if ESPHealth then
            local healthPercent = hum.Health / hum.MaxHealth
            data.healthBarBG.Visible = true
            data.healthBarBG.Size = Vector2.new(3, height)
            data.healthBarBG.Position = Vector2.new(x - 6, y)

            data.healthBar.Visible = true
            data.healthBar.Size = Vector2.new(3, height * healthPercent)
            data.healthBar.Position = Vector2.new(x - 6, y + height * (1 - healthPercent))
        else
            data.healthBarBG.Visible = false
            data.healthBar.Visible = false
        end
    end
end

RunService.RenderStepped:Connect(UpdateESP)

Players.PlayerRemoving:Connect(function(plr)
    if ESPObjects[plr] then
        for _, d in pairs(ESPObjects[plr]) do d:Remove() end
        ESPObjects[plr] = nil
    end
end)

local ESPGroup = Tabs.ESP:AddLeftGroupbox("ESP Settings")
ESPGroup:AddToggle("ESPBoxesToggle", { Text = "Show Boxes", Default = false, Callback = function(v) ESPBoxes = v end })
ESPGroup:AddToggle("ESPNamesToggle", { Text = "Show Names", Default = false, Callback = function(v) ESPNames = v end })
ESPGroup:AddToggle("ESPTracersToggle", { Text = "Show Tracers", Default = false, Callback = function(v) ESPTracers = v end })
ESPGroup:AddToggle("ESPHealthToggle", { Text = "Show Health Bars", Default = false, Callback = function(v) ESPHealth = v end })

-- =============================================
--             CHAT SPAM 2026
-- =============================================
local ChatSpamGroup = Tabs.Visual:AddLeftGroupbox("Chat Spam 2026")

local spamEnabled = false
local spamMessage = "Nesix Hub on top 🔥"
local spamDelay = 1.2

ChatSpamGroup:AddToggle("ChatSpamToggle", { Text = "Enable Chat Spam", Default = false, Callback = function(v) spamEnabled = v end })
ChatSpamGroup:AddInput("SpamMessageInput", { Text = "Spam Message", Default = "Nesix Hub on top 🔥", Placeholder = "Mensaje para spamear...", Callback = function(v) spamMessage = v end })
ChatSpamGroup:AddSlider("SpamDelaySlider", { Text = "Spam Delay (seconds)", Default = 1.2, Min = 0.8, Max = 5, Rounding = 1, Callback = function(v) spamDelay = v end })

local lastTime = 0
RunService.Heartbeat:Connect(function()
    if not spamEnabled then return end
    if tick() - lastTime < spamDelay then return end

    if spamMessage and #spamMessage > 0 then
        pcall(function()
            if TextChatService.ChatVersion == Enum.ChatVersion.TextChatService then
                local channel = TextChatService:FindFirstChild("TextChannels") and TextChatService.TextChannels:FindFirstChild("RBXGeneral")
                if channel then channel:SendAsync(spamMessage) end
            else
                local sayMsg = ReplicatedStorage:FindFirstChild("DefaultChatSystemChatEvents") and ReplicatedStorage.DefaultChatSystemChatEvents:FindFirstChild("SayMessageRequest")
                if sayMsg then sayMsg:FireServer(spamMessage, "All") end
            end
        end)
        lastTime = tick()
    end
end)

-- =============================================
--                  VISUAL - FAKE NAME
-- =============================================
local VisualGroup = Tabs.Visual:AddRightGroupbox("Fake Name")

local FakeNameEnabled = false
local FakeNameText = "NesixGod"

local function ApplyFakeNameToProfile()
    if not FakeNameEnabled then return end
    pcall(function()
        LocalPlayer.DisplayName = FakeNameText
        LocalPlayer.Name = FakeNameText
    end)
end

VisualGroup:AddToggle("FakeNameToggle", { Text = "Enable Fake Name", Default = false, Callback = function(v) FakeNameEnabled = v if v then ApplyFakeNameToProfile() end end })
VisualGroup:AddInput("FakeNameInput", { Text = "Fake Name", Default = "NesixGod", Placeholder = "Enter fake name...", Callback = function(v) FakeNameText = v if FakeNameEnabled then ApplyFakeNameToProfile() end end })

-- =============================================
--                  MISC - RAGEBOT FFA
-- =============================================
local MiscGroup = Tabs.Misc:AddLeftGroupbox("RageBot FFA")

local TPUnderEnabled = false
local DesyncEnabled = false
local UnderDistance = -3.8
local TPConnection = nil
local OriginalCFrame = nil

MiscGroup:AddToggle("TPUnderToggle", { Text = "Enable RageBot FFA", Default = false, Callback = function(v) TPUnderEnabled = v if v then SilentAimEnabled = true SilentRageEnabled = true else SilentAimEnabled = false SilentRageEnabled = false end end })
MiscGroup:AddToggle("DesyncToggle", { Text = "Desync (BETA)", Default = false, Callback = function(v) DesyncEnabled = v end })
MiscGroup:AddSlider("UnderDistanceSlider", { Text = "Distance Below Enemy", Default = 3.8, Min = 2, Max = 6, Rounding = 1, Callback = function(v) UnderDistance = -v end })

UserInputService.TouchStarted:Connect(function()
    if not TPUnderEnabled then return end
    local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if root then OriginalCFrame = root.CFrame end

    if TPConnection then TPConnection:Disconnect() end

    TPConnection = RunService.Heartbeat:Connect(function()
        local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not root then return end

        local closest, closestDist = nil, math.huge
        for _, p in pairs(Players:GetPlayers()) do
            if p == LocalPlayer then continue end
            local char = p.Character
            if not char then continue end
            local hum = char:FindFirstChildOfClass("Humanoid")
            if not hum or hum.Health <= 0 then continue end
            local targetRoot = char:FindFirstChild("HumanoidRootPart")
            if not targetRoot then continue end
            local dist = (root.Position - targetRoot.Position).Magnitude
            if dist < closestDist and dist < 400 then
                closestDist = dist
                closest = targetRoot
            end
        end

        if closest then
            root.CFrame = closest.CFrame * CFrame.new(0, UnderDistance, 0)
        end
    end)
end)

UserInputService.TouchEnded:Connect(function()
    if TPConnection then
        TPConnection:Disconnect()
        TPConnection = nil
        if DesyncEnabled and OriginalCFrame then
            local root = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            if root then root.CFrame = OriginalCFrame end
        end
    end
end)

-- =============================================
--              AUTO LOAD ON SERVER HOP (GitHub)
-- =============================================
local TeleportService = game:GetService("TeleportService")
local LocalPlayer = game.Players.LocalPlayer

-- Tu GitHub (cambia si subes el archivo con otro nombre)
local scriptUrl = "https://raw.githubusercontent.com/gn387481/NesixHub/main/NesixHub.lua"

local function AutoReExecute()
    print("🔄 Server Hop detectado - Reejecutando Nesix Hub...")
    pcall(function()
        queue_on_teleport('loadstring(game:HttpGet("' .. scriptUrl .. '"))()')
    end)
end

TeleportService.TeleportInitFailed:Connect(function(player)
    if player == LocalPlayer then
        task.wait(1.5)
        AutoReExecute()
    end
end)

print("✅ Auto Load (GitHub) activado")

-- =============================================
--               WATERMARK + UI SETTINGS
-- =============================================
Library:SetWatermarkVisibility(true)

local FrameTimer = tick()
local FrameCounter = 0
local FPS = 60
local GetPing = (function() return math.floor(game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue()) end)
local CanDoPing = pcall(function() return GetPing() end)

local WatermarkConnection = RunService.RenderStepped:Connect(function()
    FrameCounter += 1
    if (tick() - FrameTimer) >= 1 then
        FPS = FrameCounter
        FrameTimer = tick()
        FrameCounter = 0
    end
    if CanDoPing then
        Library:SetWatermark(("Nesix Hub | %d fps | %d ms"):format(math.floor(FPS), GetPing()))
    else
        Library:SetWatermark(("Nesix Hub | %d fps"):format(math.floor(FPS)))
    end
end)

local MenuGroup = Tabs["UI Settings"]:AddLeftGroupbox("Menu")
MenuGroup:AddToggle("KeybindMenuOpen", { Default = Library.KeybindFrame.Visible, Text = "Open Keybind Menu", Callback = function(v) Library.KeybindFrame.Visible = v end })
MenuGroup:AddToggle("ShowCustomCursor", { Text = "Custom Cursor", Default = true, Callback = function(v) Library.ShowCustomCursor = v end })
MenuGroup:AddDivider()
MenuGroup:AddLabel("Menu Bind"):AddKeyPicker("MenuKeybind", { Default = "RightShift", NoUI = true, Text = "Menu Keybind" })
MenuGroup:AddButton("Unload", function() Library:Unload() end)

Library.ToggleKeybind = Options.MenuKeybind

ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ "MenuKeybind" })
ThemeManager:SetFolder("NesixHub")
SaveManager:SetFolder("NesixHub")
SaveManager:BuildConfigSection(Tabs["UI Settings"])
ThemeManager:ApplyToTab(Tabs["UI Settings"])
SaveManager:LoadAutoloadConfig()

Library:OnUnload(function()
    WatermarkConnection:Disconnect()
    if fovCircle then fovCircle:Remove() end
    for _, data in pairs(ESPObjects) do
        for _, d in pairs(data) do if d.Remove then d:Remove() end end
    end
    if TPConnection then TPConnection:Disconnect() end
    print("Nesix Hub - Unloaded")
end)

print("✅ Nesix Hub Loaded Successfully!")
