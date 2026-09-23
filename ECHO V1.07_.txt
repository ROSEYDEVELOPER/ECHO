-- ECHO SAFE DATATYPE CACHE
local EchoColorFromRGB = Color3.fromRGB
local EchoColorNew = Color3.new

-- ECHO
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local UserInputService = game:GetService("UserInputService")
local ContextActionService = game:GetService("ContextActionService")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local Debris = game:GetService("Debris")
local Stats = game:GetService("Stats")
local TweenService = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")

local LocalPlayer = Players.LocalPlayer

-- Shared DestroyToy remote for scopes that do not define their own local.
local DestroyToy = nil
pcall(function()
    local MenuToys = ReplicatedStorage:FindFirstChild("MenuToys")
    DestroyToy = MenuToys and MenuToys:FindFirstChild("DestroyToy")
end)
-- ECHO EXECUTOR COMPATIBILITY LAYER
-- Compatibility only. Existing Echo layout and feature code are left untouched.
do
    local EchoG = _G

    function EchoGlobal(name)
        local ok, value = pcall(function() return EchoG[name] end)
        if ok then return value end
        return nil
    end

    -- Shared environment fallback.
    local EchoGetGenv = EchoGlobal("getgenv")
    if type(EchoGetGenv) ~= "function" then
        EchoGetGenv = function() return EchoG end
        pcall(function() EchoG.getgenv = EchoGetGenv end)
    end

    -- Compile fallback used by executors that expose either loadstring or load.
    local EchoLoadstring = EchoGlobal("loadstring") or EchoGlobal("load")
    if type(EchoLoadstring) ~= "function" then
        local ok, fn = pcall(function() return loadstring end)
        if ok and type(fn) == "function" then EchoLoadstring = fn end
    end

    -- Normalize the common request APIs used by desktop and mobile executors.
    function EchoRequest(url)
        local requestFns = {
            EchoGlobal("request"),
            EchoGlobal("http_request"),
        }

        local syn = EchoGlobal("syn")
        if type(syn) == "table" then
            table.insert(requestFns, syn.request)
        end

        for _, req in ipairs(requestFns) do
            if type(req) == "function" then
                local ok, response = pcall(function()
                    return req({
                        Url = url,
                        URL = url,
                        Method = "GET",
                        Headers = { ["User-Agent"] = "Echo" },
                    })
                end)
                if ok and type(response) == "table" then
                    local body = response.Body or response.body or response.ResponseBody
                    if type(body) == "string" and #body > 0 then
                        return body
                    end
                end
            end
        end

        return nil
    end

    function EchoHttpGet(url)
        -- Prefer the executor's native HttpGet first.
        local ok, result = pcall(function() return game:HttpGet(url) end)
        if ok and type(result) == "string" and #result > 0 then
            return result
        end

        -- Then try the request variants exposed by Delta/Volt/other executors.
        local requested = EchoRequest(url)
        if type(requested) == "string" and #requested > 0 then
            return requested
        end

        -- Last fallback: Roblox HttpService when the executor permits it.
        local httpService = game:GetService("HttpService")
        local ok2, result2 = pcall(function() return httpService:GetAsync(url) end)
        if ok2 and type(result2) == "string" and #result2 > 0 then
            return result2
        end

        error("Echo could not download: " .. tostring(url))
    end

    function EchoLoadRemote(url)
        if type(EchoLoadstring) ~= "function" then
            error("Echo requires loadstring/load support in this executor")
        end

        local source = EchoHttpGet(url)
        local chunk, err = EchoLoadstring(source)
        if type(chunk) ~= "function" then
            error(err or "Echo failed to compile downloaded code")
        end
        return chunk()
    end

    -- Clipboard compatibility. Missing clipboard APIs simply disable the copy action.
    if type(EchoGlobal("setclipboard")) ~= "function" then
        local fallbackClipboard = EchoGlobal("toclipboard")
        local syn = EchoGlobal("syn")
        if type(fallbackClipboard) == "function" then
            pcall(function() EchoG.setclipboard = fallbackClipboard end)
        elseif type(syn) == "table" and type(syn.write_clipboard) == "function" then
            pcall(function() EchoG.setclipboard = syn.write_clipboard end)
        end
    end

    -- Executor-name compatibility.
    if type(EchoGlobal("identifyexecutor")) ~= "function" then
        local getName = EchoGlobal("getexecutorname")
        if type(getName) == "function" then
            EchoG.identifyexecutor = getName
        else
            local function EchoUnknownExecutor()
                return "Unknown"
            end
            EchoG.identifyexecutor = EchoUnknownExecutor
        end
    end

    -- Touch-safe GUI parent fallback. CoreGui remains the first choice, so the layout is unchanged.
    function EchoGetGuiParent()
        local gethui = EchoGlobal("gethui")
        if type(gethui) == "function" then
            local ok, gui = pcall(gethui)
            if ok and gui then return gui end
        end

        local ok, gui = pcall(function() return game:GetService("CoreGui") end)
        if ok and gui then return gui end

        local players = game:GetService("Players")
        local player = players.LocalPlayer
        if player then
            return player:FindFirstChildOfClass("PlayerGui") or player:WaitForChild("PlayerGui")
        end

        return nil
    end

    -- Export only compatibility helpers; the Echo UI/feature implementation stays below unchanged.
    EchoG.__EchoGetGuiParent = EchoGetGuiParent

function EchoDecodeAnimeBase64(data)
    local chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
    data = data:gsub("[^%w%+/=]", "")

    local output = {}

    for i = 1, #data, 4 do
        local a = chars:find(data:sub(i, i), 1, true)
        local b = chars:find(data:sub(i + 1, i + 1), 1, true)
        local c = chars:find(data:sub(i + 2, i + 2), 1, true)
        local d = chars:find(data:sub(i + 3, i + 3), 1, true)

        a = a and (a - 1) or 0
        b = b and (b - 1) or 0
        c = c and (c - 1) or 0
        d = d and (d - 1) or 0

        local n = a * 262144 + b * 4096 + c * 64 + d

        output[#output + 1] = string.char(
            math.floor(n / 65536) % 256
        )

        if data:sub(i + 2, i + 2) ~= "=" then
            output[#output + 1] = string.char(
                math.floor(n / 256) % 256
            )
        end

        if data:sub(i + 3, i + 3) ~= "=" then
            output[#output + 1] = string.char(
                n % 256
            )
        end
    end

    return table.concat(output)
end


function EchoGetAnimeAsset(path)
    local getter = nil
    if type(getcustomasset) == "function" then
        getter = getcustomasset
    elseif type(getsynasset) == "function" then
        getter = getsynasset
    end
    if not getter then return nil end
    local ok, asset = pcall(getter, path)
    if ok and type(asset) == "string" and asset ~= "" then
        return asset
    end
    return nil
end

function EchoWriteAnimeImage(path, data)
    if type(writefile) ~= "function" then return false end
    local ok, bytes = pcall(EchoDecodeAnimeBase64, data)
    if not ok or type(bytes) ~= "string" or #bytes == 0 then return false end
    local wrote = pcall(writefile, path, bytes)
    return wrote
end

function EchoLoadAnimeImage(path, data)
    if EchoWriteAnimeImage(path, data) then
        for _ = 1, 20 do
            local asset = EchoGetAnimeAsset(path)
            if asset then return asset end
            task.wait(0.05)
        end
    end
    return EchoGetAnimeAsset(path)
end

    EchoG.__EchoHttpGet = EchoHttpGet
    EchoG.__EchoLoadRemote = EchoLoadRemote
    EchoG.__EchoLoadstring = EchoLoadstring
end

local EchoGetGenv = _G.getgenv
local EchoLoadstring = _G.__EchoLoadstring or rawget(_G, "loadstring") or rawget(_G, "load")
local EchoEnv = EchoGetGenv()
local EchoHttpGet = _G.__EchoHttpGet
local EchoLoadRemote = _G.__EchoLoadRemote

function safeValue(obj, default)
    if obj == nil then return default end
    local ok, value = pcall(function() return obj.Value end)
    if ok then return value end
    return default
end

local SelectedFood = nil
local antiInputThread = nil
local autoResetConns = {}
local autoLeaveConns = {}

local Repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"

local Library = EchoLoadRemote(Repo .. "Library.lua")
local ThemeManager = EchoLoadRemote(Repo .. "addons/ThemeManager.lua")
local SaveManager = EchoLoadRemote(Repo .. "addons/SaveManager.lua")

local Options = Library.Options
local Toggles = Library.Toggles
local Version = "v1.5"
local Logo = 139851104815795
local Discord = "https://discord.gg/QPJ9k9Yqq"

-- Keep every Obsidian notification branded as Echo.
do
    local EchoBaseNotify = Library.Notify
    if type(EchoBaseNotify) == "function" and not Library.__EchoTitleWrapped then
        Library.__EchoTitleWrapped = true
        Library.Notify = function(self, data, ...)
            if type(data) == "table" then
                if data.Title == nil or data.Title == "" then data.Title = "Echo" end
                if data.Image == nil then data.Image = "rbxassetid://" .. tostring(Logo) end
            elseif type(data) == "string" then
                data = {
                    Title = "Echo",
                    Description = data,
                    Time = 3,
                    Image = "rbxassetid://" .. tostring(Logo),
                }
            end
            return EchoBaseNotify(self, data, ...)
        end
    end
end

Library.ForceCheckbox = true
Library.ShowToggleFrameInKeybinds = true
Library.NotifyOnError = true

local Window = Library:CreateWindow({
    Title = "",
    Footer = "Echo | " .. Version,
    Icon = Logo,
    NotifySide = "Right",
    ShowCustomCursor = false,
    EnableCompacting = true,
    SidebarCompacted = false,
    CornerRadius = 18,
})

pcall(function()
    Window:SetSidebarWidth(46)
end)

pcall(function()
    Window:SetAnimations({
        ToggleWindow = true,
        TabSwitch = true,
        Groupbox = true,
        Dropdown = true,
        KeyPicker = true,
    }, 0.22, 26, "bottom")
end)

local OverlayGui = Instance.new("ScreenGui")
OverlayGui.Name = "EchoOverlay"
OverlayGui.ResetOnSpawn = false
OverlayGui.IgnoreGuiInset = true
OverlayGui.DisplayOrder = 999999
OverlayGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

pcall(function()
    OverlayGui.Parent = (_G.__EchoGetGuiParent and _G.__EchoGetGuiParent()) or CoreGui
end)

local Overlay = Instance.new("Frame")
Overlay.Name = "Overlay"
Overlay.Size = UDim2.new(0, 285, 0, 44)
Overlay.Position = UDim2.new(0, 15, 0, 15)
Overlay.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
Overlay.BackgroundTransparency = 0.12
Overlay.BorderSizePixel = 0
Overlay.Active = true
Overlay.Parent = OverlayGui

local OverlayCorner = Instance.new("UICorner")
OverlayCorner.CornerRadius = UDim.new(0, 10)
OverlayCorner.Parent = Overlay

local OverlayStroke = Instance.new("UIStroke")
OverlayStroke.Thickness = 1
OverlayStroke.Transparency = 0.45
OverlayStroke.Parent = Overlay

local LogoImage = Instance.new("ImageLabel")
LogoImage.Name = "Logo"
LogoImage.Size = UDim2.new(0, 30, 0, 30)
LogoImage.Position = UDim2.new(0, 7, 0.5, -15)
LogoImage.BackgroundTransparency = 1
LogoImage.Image = "rbxassetid://82205250816622"
LogoImage.ScaleType = Enum.ScaleType.Fit
LogoImage.Parent = Overlay

local EchoText = Instance.new("TextLabel")
EchoText.Name = "Echo"
EchoText.Size = UDim2.new(0, 55, 1, 0)
EchoText.Position = UDim2.new(0, 43, 0, 0)
EchoText.BackgroundTransparency = 1
EchoText.Text = "Echo"
EchoText.Font = Enum.Font.GothamBold
EchoText.TextSize = 15
EchoText.TextXAlignment = Enum.TextXAlignment.Left
EchoText.TextColor3 = Color3.fromRGB(235, 235, 235)
EchoText.Parent = Overlay

local Separator = Instance.new("TextLabel")
Separator.Size = UDim2.new(0, 10, 1, 0)
Separator.Position = UDim2.new(0, 91, 0, 0)
Separator.BackgroundTransparency = 1
Separator.Text = "|"
Separator.Font = Enum.Font.Gotham
Separator.TextSize = 14
Separator.TextColor3 = Color3.fromRGB(100, 100, 100)
Separator.Parent = Overlay

local PingText = Instance.new("TextLabel")
PingText.Name = "Ping"
PingText.Size = UDim2.new(0, 65, 1, 0)
PingText.Position = UDim2.new(0, 103, 0, 0)
PingText.BackgroundTransparency = 1
PingText.Text = "-- ms"
PingText.Font = Enum.Font.GothamMedium
PingText.TextSize = 13
PingText.TextXAlignment = Enum.TextXAlignment.Left
PingText.TextColor3 = Color3.fromRGB(210, 210, 210)
PingText.Parent = Overlay

local FPSLabel = Instance.new("TextLabel")
FPSLabel.Name = "FPS"
FPSLabel.Size = UDim2.new(0, 70, 1, 0)
FPSLabel.Position = UDim2.new(0, 168, 0, 0)
FPSLabel.BackgroundTransparency = 1
FPSLabel.Text = "-- FPS"
FPSLabel.Font = Enum.Font.GothamMedium
FPSLabel.TextSize = 13
FPSLabel.TextXAlignment = Enum.TextXAlignment.Left
FPSLabel.TextColor3 = Color3.fromRGB(210, 210, 210)
FPSLabel.Parent = Overlay

local dragging = false
local dragStart
local startPosition
local dragInput

function updateDrag(input)
    local delta = input.Position - dragStart
    Overlay.Position = UDim2.new(
        startPosition.X.Scale,
        startPosition.X.Offset + delta.X,
        startPosition.Y.Scale,
        startPosition.Y.Offset + delta.Y
    )
end

Overlay.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPosition = Overlay.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

Overlay.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        updateDrag(input)
    end
end)

local Frames = 0
local LastFPSUpdate = os.clock()
local CurrentFPS = 0

local FPSConnection = RunService.RenderStepped:Connect(function()
    Frames = Frames + 1
    local now = os.clock()
    if now - LastFPSUpdate >= 0.5 then
        CurrentFPS = math.floor(Frames / (now - LastFPSUpdate) + 0.5)
        Frames = 0
        LastFPSUpdate = now
        FPSLabel.Text = tostring(CurrentFPS) .. " FPS"
    end
end)

local PingConnection = task.spawn(function()
    while OverlayGui.Parent do
        local ping
        pcall(function()
            ping = Stats.Network.ServerStatsItem["Data Ping"]:GetValue()
        end)
        if ping then
            PingText.Text = math.floor(ping + 0.5) .. " ms"
        else
            PingText.Text = "-- ms"
        end
        task.wait(1)
    end
end)

pcall(function()
    Library:OnUnload(function()
        if FPSConnection then
            FPSConnection:Disconnect()
            FPSConnection = nil
        end
        if _G.EchoCleanupVisuals then
            pcall(_G.EchoCleanupVisuals)
        end
        if OverlayGui then
            OverlayGui:Destroy()
        end
    end)
end)

local Tabs = {}

Tabs.Main = Window:AddTab(" ", "house")
Tabs.Player = Window:AddTab(" ", "user-round")
Tabs.Grab = Window:AddTab("", "hand")
Tabs.Defence = Window:AddTab("", "shield-user")

local InvincibilityGroup = Tabs.Defence:AddLeftGroupbox("Invincibility", "shield-user")
local MiscellaneousGroup = Tabs.Defence:AddRightGroupbox("Miscellaneous", "boxes")
local ConfigGroup = Tabs.Defence:AddRightGroupbox("Config", "settings")

Tabs.Target = Window:AddTab("", "crosshair")
Tabs.Misc = Window:AddTab("Misc", "box")
Tabs.Figure = Window:AddTab("", "send")
Tabs.Visuals = Window:AddTab(" ", "eye")
Tabs.Server = Window:AddTab("Server", "server")
Tabs.Settings = Window:AddTab(" ", "settings")

-- ============================================================
-- MISC TAB — REBUILT (v1.7)
-- Layout: Coconut Features | Extra features | Avatar | Target Explosion | Train
-- ============================================================
do
    local MiscTab = Tabs.Misc

    -- ═══════════════════════════════════════════════════════════
    -- SHARED COCONUT CORE (real-time tracker + state)
    -- ═══════════════════════════════════════════════════════════
    local coconutTarget = nil
    local NONE_OPTION = "None (Self)"
    local COCONUT_LERP = 0.35
    local OWNER_RECLAIM_INTERVAL = 0.25

    local CoconutState = {
        PenisEnabled = false,
        BoobsEnabled = false,
        AssEnabled   = false,
        AssClapEnabled = false,

        Amount  = 10,

        BendX = 0,
        BendY = 0,

        AssClapSpeed = 2.0,
        AssClapRange = 0.5,
    }

    local CoconutTracker = {
        Root = nil,
        Position = Vector3.zero,
        CFrame = CFrame.new(),
        Velocity = Vector3.zero,
        Frame = 0,
    }

    local function getCoconutPlayerList()
        local list = { NONE_OPTION }
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                table.insert(list, plr.DisplayName .. " (@" .. plr.Name .. ")")
            end
        end
        return list
    end

    local function getCoconutPlayerFromSelection(sel)
        if not sel or sel == "" or sel == NONE_OPTION then return nil end
        local username = sel:match("@(.-)%)")
        if username then return Players:FindFirstChild(username) end
        return nil
    end

    local function getCoconutAnchorRoot()
        local target = coconutTarget
        if target and target.Parent and target.Character then
            local tRoot = target.Character:FindFirstChild("HumanoidRootPart")
            local tHum  = target.Character:FindFirstChildOfClass("Humanoid")
            if tRoot and tHum and tHum.Health > 0 then
                return tRoot
            end
        end
        local char = LocalPlayer.Character
        return char and char:FindFirstChild("HumanoidRootPart")
    end

    local function refreshCoconutTracker()
        local root = getCoconutAnchorRoot()
        CoconutTracker.Root = root
        if root then
            CoconutTracker.Position = root.Position
            CoconutTracker.CFrame   = root.CFrame
            CoconutTracker.Velocity = root.Velocity
        else
            CoconutTracker.Position = Vector3.zero
            CoconutTracker.CFrame   = CFrame.new()
            CoconutTracker.Velocity = Vector3.zero
        end
        CoconutTracker.Frame = CoconutTracker.Frame + 1
    end

    local function getCoconutFolder()
        return Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
    end

    local function collectCoconuts()
        local folder = getCoconutFolder()
        local list = {}
        if folder then
            for _, toy in ipairs(folder:GetChildren()) do
                if toy.Name == "FoodCoconut" then
                    table.insert(list, toy)
                end
            end
        end
        return list
    end

    local function setupCoconutPhysics(coconut)
        local part     = coconut:FindFirstChild("SoundPart")
        local holdPart = coconut:FindFirstChild("HoldPart")
        local rigid    = holdPart and holdPart:FindFirstChild("RigidConstraint")

        -- Mark this coconut the first time we see it. After we've sent the
        -- one-time DestroyToy the marker prevents us from ever re-firing it.
        if coconut and not coconut:GetAttribute("EchoCoconutCleaned") then
            coconut:SetAttribute("EchoCoconutCleaned", false)
        end

        return part, holdPart, rigid
    end

    local function makeCoconutInvisible(coconut)
        for _, obj in ipairs(coconut:GetChildren()) do
            if obj:IsA("BasePart") then
                obj.CanCollide  = false
                obj.CanQuery    = false
                obj.CanTouch    = false
                obj.Massless    = true
                obj.CastShadow  = false
                if obj.Transparency ~= 1 then
                    obj.Transparency = 0
                end
            end
        end
    end

  local function applyCoconutMotion(part, targetCF, reclaimDue, SetNetworkOwner, DestroyToyLocal, coconut, rigid)
        if part then
            part.CFrame                    = targetCF
            part.Velocity                  = Vector3.zero
            part.AssemblyLinearVelocity    = Vector3.zero
            part.AssemblyAngularVelocity   = Vector3.zero
            part.RotVelocity               = Vector3.zero

            if SetNetworkOwner then
                local hint = CFrame.lookAt(part.Position, targetCF.Position)
                pcall(function() SetNetworkOwner:FireServer(part, hint) end)
            end
        end

        if rigid and DestroyToyLocal and coconut then
            local alreadyCleaned = coconut:GetAttribute("EchoCoconutCleaned")
            if not alreadyCleaned then
                pcall(function()
                    if rigid.Enabled and rigid.Attachment1 then
                        DestroyToyLocal:FireServer(coconut)
                        coconut:SetAttribute("EchoCoconutCleaned", true)
                    end
                end)
            end
        end
    end

    -- ─────────────────────────────────────────────────────────
    -- Coconut Penis — chain of N coconuts
    -- ─────────────────────────────────────────────────────────
    local coconutPenisThread = nil

    local function startCoconutPenis()
        if coconutPenisThread then return end
        coconutPenisThread = task.spawn(function()
            local RS = game:GetService("ReplicatedStorage")
            local SetNetworkOwner = RS.GrabEvents.SetNetworkOwner
            local SpawnToy        = RS.MenuToys.SpawnToyRemoteFunction
            local DestroyToyLocal = RS.MenuToys.DestroyToy

            local SEGMENT_LENGTH     = 1.3
            local PER_SEGMENT_DEG    = 25
            local lastOwnerReclaim   = 0

            while CoconutState.PenisEnabled do
                refreshCoconutTracker()
                local root = CoconutTracker.Root
                if not root then
                    RunService.RenderStepped:Wait()
                    continue
                end

                local now = tick()
                local reclaimDue = (now - lastOwnerReclaim) > OWNER_RECLAIM_INTERVAL
                if reclaimDue then lastOwnerReclaim = now end

                local coconuts = collectCoconuts()
                local required = CoconutState.Amount + 2

                if #coconuts < required then
                    task.spawn(function()
                        SpawnToy:InvokeServer("FoodCoconut", root.CFrame * CFrame.new(-5, 0, 10), Vector3.zero)
                    end)
                end

                local velocityComp  = CFrame.new(root.Velocity / 100)
                local bendX         = CoconutState.BendX
                local bendY         = CoconutState.BendY

                local stepYawRad   = math.rad(-bendX * PER_SEGMENT_DEG)
                local stepPitchRad = math.rad(-bendY * PER_SEGMENT_DEG)

                local baseAnchor = root.CFrame
                    * CFrame.new(0, -1, -0.5)
                    * velocityComp

                local shaftCount = math.max(0, #coconuts - 2)
                local segmentCFrames = table.create(shaftCount)
                local acc = baseAnchor
                for k = 1, shaftCount do
                    segmentCFrames[k] = acc
                    acc = acc
                        * CFrame.new(0, 0, -SEGMENT_LENGTH)
                        * CFrame.Angles(stepPitchRad, stepYawRad, 0)
                end

                for i, coconut in ipairs(coconuts) do
                    local part, holdPart, rigid = setupCoconutPhysics(coconut)
                    if part and holdPart and rigid then
                        local targetCF
                        if i == 1 then
                            targetCF = root.CFrame * CFrame.new(-0.35, -1.0, -0.2) * velocityComp
                        elseif i == 2 then
                            targetCF = root.CFrame * CFrame.new(0.35, -1.0, -0.2) * velocityComp
                        else
                            local k = i - 2
                            targetCF = segmentCFrames[k] or baseAnchor
                        end

                        applyCoconutMotion(part, targetCF, reclaimDue,
                            SetNetworkOwner, DestroyToyLocal, coconut, rigid)
                        makeCoconutInvisible(coconut)
                    end
                end

                RunService.RenderStepped:Wait()
            end
            coconutPenisThread = nil
        end)
    end

    local function stopCoconutPenis()
        CoconutState.PenisEnabled = false
        if coconutPenisThread then
            pcall(task.cancel, coconutPenisThread)
            coconutPenisThread = nil
        end
    end

    -- ─────────────────────────────────────────────────────────
    -- Coconut Boobs — 2 coconuts on chest only
    -- ─────────────────────────────────────────────────────────
    local coconutBoobsThread = nil

    local function startCoconutBoobs()
        if coconutBoobsThread then return end
        coconutBoobsThread = task.spawn(function()
            local RS = game:GetService("ReplicatedStorage")
            local SetNetworkOwner = RS.GrabEvents.SetNetworkOwner
            local SpawnToy        = RS.MenuToys.SpawnToyRemoteFunction
            local DestroyToyLocal = RS.MenuToys.DestroyToy

            local lastOwnerReclaim = 0

            while CoconutState.BoobsEnabled do
                refreshCoconutTracker()
                local root = CoconutTracker.Root
                if not root then
                    RunService.RenderStepped:Wait()
                    continue
                end

                local now = tick()
                local reclaimDue = (now - lastOwnerReclaim) > OWNER_RECLAIM_INTERVAL
                if reclaimDue then lastOwnerReclaim = now end

                local coconuts = collectCoconuts()

                if #coconuts < 2 then
                    for _ = 1, 2 - #coconuts do
                        task.spawn(function()
                            SpawnToy:InvokeServer("FoodCoconut", root.CFrame * CFrame.new(-5, 0, 10), Vector3.zero)
                        end)
                    end
                end

                for i = 1, math.min(2, #coconuts) do
                    local coconut = coconuts[i]
                    local part, holdPart, rigid = setupCoconutPhysics(coconut)
                    if part and holdPart and rigid then
                        local targetCF
                        if i == 1 then
                            targetCF = root.CFrame * CFrame.new(-0.4, 0.3, -0.55)
                        else
                            targetCF = root.CFrame * CFrame.new(0.4, 0.3, -0.55)
                        end
                        local smoothCF = part.CFrame:Lerp(targetCF, COCONUT_LERP)
                        applyCoconutMotion(part, smoothCF, reclaimDue,
                            SetNetworkOwner, DestroyToyLocal, coconut, rigid)
                        makeCoconutInvisible(coconut)
                    end
                end

                RunService.RenderStepped:Wait()
            end
            coconutBoobsThread = nil
        end)
    end

    local function stopCoconutBoobs()
        CoconutState.BoobsEnabled = false
        if coconutBoobsThread then
            pcall(task.cancel, coconutBoobsThread)
            coconutBoobsThread = nil
        end
    end

    -- ─────────────────────────────────────────────────────────
    -- Coconut Ass — 2 coconuts on cheeks only
    -- ─────────────────────────────────────────────────────────
    local coconutAssThread = nil

    local function startCoconutAss()
        if coconutAssThread then return end
        coconutAssThread = task.spawn(function()
            local RS = game:GetService("ReplicatedStorage")
            local SetNetworkOwner = RS.GrabEvents.SetNetworkOwner
            local SpawnToy        = RS.MenuToys.SpawnToyRemoteFunction
            local DestroyToyLocal = RS.MenuToys.DestroyToy

            local BASE_X = 0.35
            local phase  = 0

            while CoconutState.AssEnabled do
                refreshCoconutTracker()
                local root = CoconutTracker.Root
                if not root then
                    RunService.RenderStepped:Wait()
                    continue
                end

                local coconuts = collectCoconuts()

                -- Ass always uses the LAST 2 coconuts in the folder, so it
                -- never fights Boobs (which uses the first 2) for slots.
                -- If Penis is also on, its shaft coconuts are in the middle.
                local total = #coconuts
                local assStart = total - 1   -- 1-based index of first cheek
                if assStart < 1 then assStart = 1 end

                local needed = 2
                if total < needed then
                    for _ = 1, needed - total do
                        task.spawn(function()
                            SpawnToy:InvokeServer("FoodCoconut", root.CFrame * CFrame.new(-5, 0, 10), Vector3.zero)
                        end)
                    end
                    RunService.RenderStepped:Wait()
                    continue
                end

                local swing = 0
                local dt = RunService.RenderStepped:Wait()
                if CoconutState.AssClapEnabled then
                    local range = CoconutState.AssClapRange
                    phase = phase + (CoconutState.AssClapSpeed * dt * math.pi * 2)
                    swing = math.sin(phase) * range
                else
                    phase = phase * 0.85
                    if math.abs(phase) < 0.01 then phase = 0 end
                end

                -- Grab the two cheek coconuts (last 2 in the folder).
                local cheeks = { coconuts[assStart], coconuts[assStart + 1] }

                for i = 1, 2 do
                    local coconut = cheeks[i]
                    if coconut then
                        local part, holdPart, rigid = setupCoconutPhysics(coconut)
                        if part and holdPart then
                            -- rigid is intentionally not required here; setup
                            -- already disabled it above.
                            local sideSign = (i == 1) and -1 or 1
                            local xOffset  = BASE_X * sideSign + swing * sideSign
                            local targetCF = root.CFrame * CFrame.new(xOffset, -1.1, 0.45)

                            local finalCF
                            if CoconutState.AssClapEnabled then
                                finalCF = targetCF
                            else
                                finalCF = part.CFrame:Lerp(targetCF, 0.25)
                            end

                            applyCoconutMotion(part, finalCF, false,
                                SetNetworkOwner, DestroyToyLocal, coconut, rigid)
                            makeCoconutInvisible(coconut)
                        end
                    end
                end
            end
            coconutAssThread = nil
        end)
    end

    local function stopCoconutAss()
        CoconutState.AssEnabled = false
        if coconutAssThread then
            pcall(task.cancel, coconutAssThread)
            coconutAssThread = nil
        end
    end

    -- ─────────────────────────────────────────────────────────
    -- Coconut 67 — static 6/7 pattern (13 coconuts)
    -- ─────────────────────────────────────────────────────────
    local Coconut67State = { Enabled = false, Thread = nil }

    local function stopCoconut67()
        Coconut67State.Enabled = false
        _G.Coconut67 = false
        if Coconut67State.Thread then
            pcall(task.cancel, Coconut67State.Thread)
            Coconut67State.Thread = nil
        end
    end

    local function startCoconut67()
        if Coconut67State.Thread then return end
        Coconut67State.Enabled = true
        _G.Coconut67 = true
        Coconut67State.Thread = task.spawn(function()
            local Me = game.Players.LocalPlayer
            local RS = game:GetService("ReplicatedStorage")
            local SetNetworkOwner = RS.GrabEvents.SetNetworkOwner
            local SpawnToy        = RS.MenuToys.SpawnToyRemoteFunction
            local DestroyToyLocal = RS.MenuToys.DestroyToy

            local pattern6 = {
                Vector3.new( 0.8,  1.6, 0), Vector3.new(-0.1,  1.8, 0),
                Vector3.new(-0.9,  1.4, 0), Vector3.new(-1.0,  0.5, 0),
                Vector3.new(-0.6, -0.4, 0), Vector3.new(-0.7, -1.0, 0),
                Vector3.new( 0.3, -1.1, 0), Vector3.new( 0.5, -0.4, 0),
            }
            local pattern7 = {
                Vector3.new(-0.9,  1.4, 0), Vector3.new(-0.1,  1.4, 0),
                Vector3.new( 0.7,  1.4, 0), Vector3.new( 0.2,  0.2, 0),
                Vector3.new(-0.4, -0.8, 0),
            }

            local totalPoints, scale = 13, 1.0
            local sixOffset, sevenOffset = -2.4, 2.4
            local lastOwnerReclaim = 0

            while Coconut67State.Enabled do
                refreshCoconutTracker()
                local refRoot = CoconutTracker.Root
                if not refRoot then
                    RunService.RenderStepped:Wait()
                    continue
                end

                local now = tick()
                local reclaimDue = (now - lastOwnerReclaim) > OWNER_RECLAIM_INTERVAL
                if reclaimDue then lastOwnerReclaim = now end

                local inv = workspace:FindFirstChild(Me.Name .. "SpawnedInToys")
                if not inv then
                    RunService.RenderStepped:Wait()
                    continue
                end

                local Coconuts = {}
                for _, toy in pairs(inv:GetChildren()) do
                    if toy.Name == "FoodCoconut" then
                        table.insert(Coconuts, toy)
                    end
                end

                while #Coconuts > totalPoints do
                    local extra = table.remove(Coconuts)
                    pcall(function() DestroyToyLocal:FireServer(extra) end)
                end

                if #Coconuts < totalPoints then
                    local missing = totalPoints - #Coconuts
                    for _ = 1, math.min(3, missing) do
                        task.spawn(function()
                            pcall(function()
                                SpawnToy:InvokeServer("FoodCoconut", refRoot.CFrame * CFrame.new(0, 20, 0), Vector3.zero)
                            end)
                        end)
                    end
                    RunService.RenderStepped:Wait()
                end

                local base  = refRoot.CFrame * CFrame.new(0, 2, 2)
                local bendX = CoconutState.BendX
                local bendY = CoconutState.BendY

                for i, Coco in ipairs(Coconuts) do
                    local Part     = Coco:FindFirstChild("SoundPart")
                    local HoldPart = Coco:FindFirstChild("HoldPart")
                    local Rigid    = HoldPart and HoldPart:FindFirstChild("RigidConstraint")

                    if Part then
                        local localOffset
                        if i <= 8 then
                            local q = pattern6[i]
                            localOffset = Vector3.new((q.X + sixOffset) * scale, q.Y * scale, 0)
                        elseif i <= 13 then
                            local q = pattern7[i - 8]
                            localOffset = Vector3.new((q.X + sevenOffset) * scale, q.Y * scale, 0)
                        else
                            localOffset = Vector3.new(0, -20, 0)
                        end

                        local bendOffset = Vector3.new(bendX * i, bendY * i, 0)
                        local targetCF   = base * CFrame.new(localOffset + bendOffset)
                        local smoothCF   = Part.CFrame:Lerp(targetCF, COCONUT_LERP)

                        applyCoconutMotion(Part, smoothCF, reclaimDue,
                            SetNetworkOwner, DestroyToyLocal, Coco, Rigid)

                        for _, part in pairs(Coco:GetChildren()) do
                            if part:IsA("BasePart") then
                                part.CanCollide, part.CanTouch, part.CanQuery = false, false, false
                                if part.Transparency ~= 1 then part.Transparency = 0 end
                            end
                        end
                    end
                end

                RunService.RenderStepped:Wait()
            end
            Coconut67State.Thread = nil
        end)
    end

    -- ─────────────────────────────────────────────────────────
    -- Coconut 67 Animation — 16 coconuts bobbing 6/7 layout
    -- ─────────────────────────────────────────────────────────
    local Coconut67AnimState = { Enabled = false, Thread = nil }

    local function stopCoconut67Animation()
        Coconut67AnimState.Enabled = false
        _G.EchoCoconut67Animation = false
        if Coconut67AnimState.Thread then
            pcall(task.cancel, Coconut67AnimState.Thread)
            Coconut67AnimState.Thread = nil
        end
    end

    local function startCoconut67Animation()
        if Coconut67AnimState.Thread then return end
        Coconut67AnimState.Enabled = true
        _G.EchoCoconut67Animation = true
        Coconut67AnimState.Thread = task.spawn(function()
            local Me = game.Players.LocalPlayer
            local RS = game:GetService("ReplicatedStorage")
            local SetNetworkOwner = RS.GrabEvents.SetNetworkOwner
            local SpawnToy        = RS.MenuToys.SpawnToyRemoteFunction
            local DestroyToyLocal = RS.MenuToys.DestroyToy

            local pattern6 = {
                Vector3.new(-0.8,  2.0, 0), Vector3.new( 0.0,  2.0, 0), Vector3.new( 0.8,  2.0, 0),
                Vector3.new(-0.8,  0.6, 0), Vector3.new(-0.8, -0.8, 0), Vector3.new( 0.0, -0.8, 0),
                Vector3.new( 0.8, -0.8, 0), Vector3.new(-0.8, -2.2, 0), Vector3.new( 0.0, -2.2, 0),
                Vector3.new( 0.8, -2.2, 0),
            }
            local pattern7 = {
                Vector3.new(-0.8,  1.6, 0), Vector3.new( 0.0,  1.6, 0), Vector3.new( 0.8,  1.6, 0),
                Vector3.new( 0.4,  0.4, 0), Vector3.new( 0.0, -0.6, 0), Vector3.new(-0.4, -1.6, 0),
            }

            local totalPoints = 16
            local scale       = 1.0
            local sixOffset   = -2.5
            local sevenOffset =  2.5

            local BOB_AMPLITUDE = 1.6
            local BOB_SPEED     = 3
            local bobTime       = 0

            while Coconut67AnimState.Enabled do
                local refRoot
                if coconutTarget and coconutTarget.Character then
                    refRoot = coconutTarget.Character:FindFirstChild("HumanoidRootPart")
                end
                if not refRoot then
                    local Char = Me.Character
                    refRoot = Char and Char:FindFirstChild("HumanoidRootPart")
                end
                if not refRoot then task.wait(0.1) continue end

                local inv = workspace:FindFirstChild(Me.Name .. "SpawnedInToys")
                if not inv then task.wait(0.1) continue end

                local Coconuts = {}
                for _, toy in pairs(inv:GetChildren()) do
                    if toy.Name == "FoodCoconut" then
                        table.insert(Coconuts, toy)
                    end
                end

                while #Coconuts > totalPoints do
                    local extra = table.remove(Coconuts)
                    pcall(function() DestroyToyLocal:FireServer(extra) end)
                end

                if #Coconuts < totalPoints then
                    local missing = totalPoints - #Coconuts
                    for _ = 1, math.min(3, missing) do
                        task.spawn(function()
                            pcall(function()
                                SpawnToy:InvokeServer("FoodCoconut", refRoot.CFrame * CFrame.new(0, 20, 0), Vector3.zero)
                            end)
                        end)
                    end
                    task.wait(0.05)
                end

                local base = refRoot.CFrame * CFrame.new(0, 2, 2)

                bobTime = bobTime + 0.05 * BOB_SPEED
                local bob6 =  math.sin(bobTime) * BOB_AMPLITUDE
                local bob7 = -math.sin(bobTime) * BOB_AMPLITUDE

                for i, Coco in ipairs(Coconuts) do
                    local Part      = Coco:FindFirstChild("SoundPart")
                    local HoldPart  = Coco:FindFirstChild("HoldPart")
                    local Rigid     = HoldPart and HoldPart:FindFirstChild("RigidConstraint")
                    local PartOwner = Part and Part:FindFirstChild("PartOwner")

                    if Part then
                        if not PartOwner or PartOwner.Value ~= Me.Name then
                            SetNetworkOwner:FireServer(Part, Part.CFrame)
                        else
                            local localOffset
                            if i <= 10 then
                                local p = pattern6[i]
                                localOffset = Vector3.new((p.X + sixOffset) * scale, p.Y * scale + bob6, 0)
                            elseif i <= 16 then
                                local p = pattern7[i - 10]
                                localOffset = Vector3.new((p.X + sevenOffset) * scale, p.Y * scale + bob7, 0)
                            else
                                localOffset = Vector3.new(0, -20, 0)
                            end

                            Part.CFrame = base * CFrame.new(localOffset)
                            Part.Velocity = Vector3.zero
                            Part.AssemblyLinearVelocity  = Vector3.zero
                            Part.AssemblyAngularVelocity = Vector3.zero
                        end

                        for _, part in pairs(Coco:GetChildren()) do
                            if part:IsA("BasePart") then
                                part.CanCollide = false
                                part.CanTouch  = false
                                part.CanQuery  = false
                                if part.Transparency ~= 1 then
                                    part.Transparency = 0
                                end
                            end
                        end

                        if Rigid and Rigid.Attachment1 then
                            DestroyToyLocal:FireServer(Coco)
                        end
                    end
                end

                task.wait(0.05)
            end
            Coconut67AnimState.Thread = nil
        end)
    end

    -- ═══════════════════════════════════════════════════════════
    -- LEFT GROUPBOX — "Coconut Features"
    -- ═══════════════════════════════════════════════════════════
    local CoconutGroup = MiscTab:AddLeftGroupbox("Coconut Features", "box")

    local CoconutTargetDropdown = CoconutGroup:AddDropdown("EchoCoconutTarget", {
        Text = "Coconut Target",
        Values = getCoconutPlayerList(),
        Default = NONE_OPTION,
        Searchable = true,
        Multi = false,
        Callback = function(v)
            coconutTarget = getCoconutPlayerFromSelection(v)
            if not coconutTarget then
                Library:Notify({ Description = "Coconut target cleared — coconuts on you", Time = 2 })
            else
                Library:Notify({ Description = "Coconut target: " .. coconutTarget.DisplayName, Time = 2 })
            end
        end,
    })

    local function refreshCoconutDropdown()
        if CoconutTargetDropdown then
            pcall(function() CoconutTargetDropdown:SetValues(getCoconutPlayerList()) end)
        end
        if coconutTarget and not coconutTarget.Parent then
            coconutTarget = nil
        end
    end

    Players.PlayerAdded:Connect(function() task.wait(0.5); refreshCoconutDropdown() end)
    Players.PlayerRemoving:Connect(function() task.wait(0.5); refreshCoconutDropdown() end)
    task.spawn(function()
        task.wait(1)
        refreshCoconutDropdown()
    end)

    CoconutGroup:AddToggle("EchoCoconutPenis", {
        Text = "Coconut Penis",
        Default = false,
        Tooltip = "Long straight line of coconuts growing forward from the pelvis",
        Callback = function(Value)
            CoconutState.PenisEnabled = Value
            if Value then startCoconutPenis() else stopCoconutPenis() end
        end,
    })

    CoconutGroup:AddToggle("EchoCoconutBoobs", {
        Text = "Coconut Boobs",
        Default = false,
        Tooltip = "Places 2 coconuts on the chest",
        Callback = function(Value)
            CoconutState.BoobsEnabled = Value
            if Value then startCoconutBoobs() else stopCoconutBoobs() end
        end,
    })

    CoconutGroup:AddToggle("EchoCoconutAss", {
        Text = "Coconut Ass",
        Default = false,
        Tooltip = "Places 2 coconuts on the cheeks",
        Callback = function(Value)
            CoconutState.AssEnabled = Value
            if Value then
                startCoconutAss()
            else
                CoconutState.AssClapEnabled = false
                if Toggles.EchoCoconutAssClap then
                    pcall(function() Toggles.EchoCoconutAssClap:SetValue(false) end)
                end
                stopCoconutAss()
            end
        end,
    })

    CoconutGroup:AddSlider("EchoCoconutAmount", {
        Text = "Coconut amount",
        Default = 10,
        Min = 1,
        Max = 25,
        Rounding = 0,
        Callback = function(Value) CoconutState.Amount = Value end,
    })

    CoconutGroup:AddSlider("EchoCoconutBendX", {
        Text = "Curve Coconut X",
        Default = 0,
        Min = -2,
        Max = 2,
        Rounding = 2,
        Tooltip = "Curves the coconut line left / right",
        Callback = function(Value) CoconutState.BendX = Value end,
    })

    CoconutGroup:AddSlider("EchoCoconutBendY", {
        Text = "Curve Coconut Y",
        Default = 0,
        Min = -2,
        Max = 2,
        Rounding = 2,
        Tooltip = "Curves the coconut line up / down",
        Callback = function(Value) CoconutState.BendY = Value end,
    })

    -- ═══════════════════════════════════════════════════════════
    -- COCONUT FEATURES — Extra features tabbox (inside the groupbox)
    -- ═══════════════════════════════════════════════════════════
    local ExtraBox = CoconutGroup:AddTabbox("Extra features")
    local ExtraTab = ExtraBox:AddTab("Extra features", "sparkles")

    ExtraTab:AddToggle("EchoCoconut67Animation", {
        Text = "Coconut 67 Animation",
        Default = false,
        Tooltip = "Animates the 6-7 coconut pattern with a bobbing motion",
        Callback = function(Value)
            if Value then startCoconut67Animation() else stopCoconut67Animation() end
        end,
    })

    ExtraTab:AddToggle("Coconut67", {
        Text = "Coconut 67",
        Default = false,
        Flag = "Coconut67",
        Tooltip = "Spawns 13 coconuts in a 6-7 pattern (follows your Coconut Target)",
        Callback = function(Value)
            if Value then startCoconut67() else stopCoconut67() end
        end,
    })

    ExtraTab:AddToggle("EchoCoconutAssClap", {
        Text = "Coconut Ass clap (client)",
        Default = false,
        Tooltip = "Makes the ass cheeks swing outward and clap back together",
        Callback = function(Value)
            if Value and not CoconutState.AssEnabled then
                Library:Notify({
                    Title = "Echo",
                    Description = "Enable Coconut Ass first",
                    Time = 3,
                })
                if Toggles.EchoCoconutAssClap then
                    pcall(function() Toggles.EchoCoconutAssClap:SetValue(false) end)
                end
                return
            end
            CoconutState.AssClapEnabled = Value and true or false
        end,
    })

    ExtraTab:AddSlider("EchoCoconutAssClapSpeed", {
        Text = "Ass clap speed",
        Default = 2.0,
        Min = 0.1,
        Max = 10,
        Rounding = 2,
        Suffix = " /s",
        Tooltip = "How many claps per second",
        Callback = function(Value) CoconutState.AssClapSpeed = Value end,
    })

    -- ═══════════════════════════════════════════════════════════
    -- LEFT GROUPBOX — "Avatar" (moved here unchanged)
    -- ═══════════════════════════════════════════════════════════
    local AvatarGroup = MiscTab:AddLeftGroupbox("Avatar", "person-standing")

    local AvatarState = {
        Korblox   = false,
        Headless  = false,
        Both      = false,
        Reapply   = false,
        Original  = nil,
    }

    local KORBLOX_LEG   = "139607718"
    local HEADLESS_HEAD = "134082579"

    local function getAvatarHumanoid()
        local char = LocalPlayer.Character
        return char and char:FindFirstChildOfClass("Humanoid")
    end

    local function captureOriginal(humanoid)
        if AvatarState.Original then return end
        local ok, desc = pcall(function() return humanoid:GetAppliedDescription() end)
        if ok and desc then
            AvatarState.Original = desc:Clone()
        end
    end

    local function applyAvatar()
        local humanoid = getAvatarHumanoid()
        if not humanoid then return end
        captureOriginal(humanoid)
        if not AvatarState.Original then return end

        local desc = AvatarState.Original:Clone()
        local useKorblox   = AvatarState.Korblox or AvatarState.Both
        local useHeadless  = AvatarState.Headless or AvatarState.Both
        if useKorblox then desc.RightLeg = KORBLOX_LEG end
        if useHeadless then desc.Head = HEADLESS_HEAD end

        pcall(function() humanoid:ApplyDescription(desc) end)
    end

    local function refreshAvatar()
        AvatarState.Original = nil
        task.wait(0.5)
        applyAvatar()
    end

    AvatarGroup:AddToggle("EchoKorblox", {
        Text = "Korblox",
        Default = false,
        Callback = function(Value)
            AvatarState.Korblox = Value and true or false
            AvatarState.Both = false
            applyAvatar()
        end,
    })

    AvatarGroup:AddToggle("EchoHeadless", {
        Text = "Headless",
        Default = false,
        Callback = function(Value)
            AvatarState.Headless = Value and true or false
            AvatarState.Both = false
            applyAvatar()
        end,
    })

    AvatarGroup:AddToggle("EchoKorbloxHeadless", {
        Text = "Both",
        Default = false,
        Callback = function(Value)
            AvatarState.Both = Value and true or false
            if Value then
                AvatarState.Korblox = false
                AvatarState.Headless = false
            end
            applyAvatar()
        end,
    })

    AvatarGroup:AddToggle("EchoAvatarReapply", {
        Text = "Reapply both on respawn",
        Default = false,
        Callback = function(Value)
            AvatarState.Reapply = Value and true or false
        end,
    })

    LocalPlayer.CharacterAdded:Connect(function()
        AvatarState.Original = nil
        if AvatarState.Reapply and (AvatarState.Korblox or AvatarState.Headless or AvatarState.Both) then
            task.spawn(refreshAvatar)
        end
    end)

    -- ═══════════════════════════════════════════════════════════
    -- RIGHT GROUPBOX — "Target Explosion" (with ViewportFrame preview)
    -- ═══════════════════════════════════════════════════════════
    local TargetExplosionGroup = MiscTab:AddRightGroupbox("Target Explosion", "bomb")

    local TargetExplosionState = {
        Enabled = false,
        Type    = "Missile",
        Amount  = 3,
        Target  = nil,
        Thread  = nil,
    }

    local ExplosionTypes = {
        Missile         = "BombMissile",
        Firework        = "FireworkMissile",
        Void            = "BombDarkMatter",
        Balloon         = "BombBalloon",
        ["Small Present"] = "PresentSmall",
        ["Big Present"]   = "PresentBig",
        Snowball        = "Snowball",
    }

    local ExplosionHitboxes = {
        BombMissile     = "PartHitDetector",
        BombDarkMatter  = "PartHitDetector",
        FireworkMissile = "PartHitDetector",
        BombBalloon     = "Balloon",
        PresentBig      = "Box",
        PresentSmall    = "Box",
        Snowball        = "Snowball",
    }

    local ExplosionSetupParts = {
        BombMissile     = "Body",
        BombDarkMatter  = "Pyramid",
        FireworkMissile = "Hitbox",
        BombBalloon     = "Balloon",
        PresentBig      = "Box",
        PresentSmall    = "Box",
        Snowball        = "Snowball",
    }

    -- ── ViewportFrame preview ─────────────────────────────────
    local PreviewViewport = Instance.new("ViewportFrame")
    PreviewViewport.Name = "EchoExplosionPreview"
    PreviewViewport.Size = UDim2.new(1, -12, 0, 140)
    PreviewViewport.Position = UDim2.new(0, 6, 0, 14)   -- ↓ push down
    PreviewViewport.BackgroundColor3 = Color3.fromRGB(22, 22, 22)
    PreviewViewport.BackgroundTransparency = 0.15
    PreviewViewport.BorderSizePixel = 0
    PreviewViewport.Ambient = Color3.fromRGB(180, 180, 180)
    PreviewViewport.LightColor = Color3.fromRGB(255, 255, 255)
    PreviewViewport.LightDirection = Vector3.new(-0.4, -0.8, -0.6)

    local PreviewCorner = Instance.new("UICorner")
    PreviewCorner.CornerRadius = UDim.new(0, 8)
    PreviewCorner.Parent = PreviewViewport

    local PreviewStroke = Instance.new("UIStroke")
    PreviewStroke.Thickness = 1
    PreviewStroke.Transparency = 0.7
    PreviewStroke.Parent = PreviewViewport

    local PreviewCamera = Instance.new("Camera")
    PreviewCamera.FieldOfView = 40
    PreviewCamera.Parent = PreviewViewport
    PreviewViewport.CurrentCamera = PreviewCamera

    -- Hidden world camera used by ViewportFrame parents that don't allow GUI cameras
    local PreviewWorldCamera = Instance.new("Camera")
    PreviewWorldCamera.Name = "EchoExplosionPreviewCamera"

    -- Put the viewport inside the groupbox as a raw element by wrapping it in a Frame.
    -- Extra top padding + a small vertical spacer pushes the preview down a bit
    -- so it doesn't crowd the groupbox title.
    local PreviewTopPad = Instance.new("Frame")
    PreviewTopPad.Name = "EchoExplosionPreviewTopPad"
    PreviewTopPad.BackgroundTransparency = 1
    PreviewTopPad.Size = UDim2.new(1, 0, 0, 14)
    PreviewTopPad.Parent = TargetExplosionGroup.Frame or TargetExplosionGroup.Container or TargetExplosionGroup

    local PreviewHolder = Instance.new("Frame")
    PreviewHolder.Name = "EchoExplosionPreviewHolder"
    PreviewHolder.BackgroundTransparency = 1
    PreviewHolder.Size = UDim2.new(1, 0, 0, 150)
    PreviewHolder.Position = UDim2.new(0, 0, 0, 14)
    PreviewHolder.Parent = TargetExplosionGroup.Frame or TargetExplosionGroup.Container or TargetExplosionGroup
    PreviewViewport.Parent = PreviewHolder

    -- We rely on Obsidian's groupbox ScrollFrame; if the groupbox doesn't expose
    -- a frame, we add the viewport to the misc tab's container directly.
    -- (Best-effort placement; if this fails, the viewport simply won't show.)

    -- Preview rotation state
    local previewYaw   = 30
    local previewPitch = 15
    local previewDragging = false
    local previewLastMouse = Vector2.zero
    local previewAutoSpin = true
    local previewModel = nil
    local previewHolderPart = nil
    local previewDistance = 8   -- dynamic, updated by loadPreviewModel

    local function computePreviewCFrame()
        local center = Vector3.new(0, 0, 0)
        local dist = previewDistance or 8
        local yawRad   = math.rad(previewYaw)
        local pitchRad = math.rad(previewPitch)
        local x = math.cos(pitchRad) * math.sin(yawRad) * dist
        local y = math.sin(pitchRad) * dist
        local z = math.cos(pitchRad) * math.cos(yawRad) * dist
        return CFrame.lookAt(center + Vector3.new(x, y, z), center)
    end

    local function clearPreviewModel()
        if previewModel and previewModel.Parent then
            previewModel:Destroy()
        end
        previewModel = nil
        if previewHolderPart and previewHolderPart.Parent then
            previewHolderPart:Destroy()
        end
        previewHolderPart = nil
    end

    -- Token so rapid dropdown changes don't race and stack previews.
    local previewLoadToken = 0

    local function computePreviewPivot(model)
        -- Center the model at the origin by using its bounding box center.
        local ok, cf, size = pcall(function()
            return model:GetBoundingBox()
        end)
        if ok and cf then
            return cf, size
        end
        return CFrame.new(0, 0, 0), Vector3.new(1, 1, 1)
    end

    local function fitCameraToModel(model)
        local centerCF, size = computePreviewPivot(model)
        local maxExtent = math.max(size.X, size.Y, size.Z)
        -- Pull the camera back based on model size so it always fits nicely.
        local dist = math.max(maxExtent * 2.2, 5)
        previewDistance = dist
    end

    local function loadPreviewModel(toyName)
        if not toyName then return end

        previewLoadToken = previewLoadToken + 1
        local token = previewLoadToken

        clearPreviewModel()

        local menu = ReplicatedStorage:FindFirstChild("MenuToys")
        if not menu then return end

        local spawnRemote  = menu:FindFirstChild("SpawnToyRemoteFunction")
        local destroyRemote = menu:FindFirstChild("DestroyToy")
        if not spawnRemote then return end

        local folder = Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        if not folder then
            task.wait(0.1)
            folder = Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        end
        if not folder then return end

        -- Snapshot existing children so we can detect the newly spawned model.
        local before = {}
        for _, child in ipairs(folder:GetChildren()) do
            before[child] = true
        end

        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        local spawnCF = hrp and (hrp.CFrame * CFrame.new(0, 5000, 0)) or CFrame.new(0, 5000, 0)

        -- Ask the server to spawn one at a hidden location.
        pcall(function()
            spawnRemote:InvokeServer(toyName, spawnCF, Vector3.zero)
        end)

        -- Wait up to ~1.5s for the new model to appear.
        local spawned = nil
        local deadline = tick() + 1.5
        while tick() < deadline do
            task.wait(0.03)
            if previewLoadToken ~= token then
                -- Another request came in; abandon this one.
                if spawned and spawned.Parent and destroyRemote then
                    pcall(function() destroyRemote:FireServer(spawned) end)
                end
                return
            end

            for _, child in ipairs(folder:GetChildren()) do
                if not before[child] and child.Name == toyName and child:IsA("Model") then
                    spawned = child
                    break
                end
            end
            if spawned then break end
        end

        if not spawned then
            -- Fallback placeholder so the viewport isn't empty.
            local placeholder = Instance.new("Part")
            placeholder.Name = "PreviewPlaceholder"
            placeholder.Size = Vector3.new(2, 2, 2)
            placeholder.Shape = Enum.PartType.Ball
            placeholder.Anchored = true
            placeholder.CanCollide = false
            placeholder.CanTouch = false
            placeholder.CanQuery = false
            placeholder.Material = Enum.Material.Neon
            placeholder.Color = Color3.fromRGB(255, 120, 60)
            placeholder.Parent = PreviewViewport
            previewModel = placeholder
            return
        end

        -- Build a static clone for the viewport.
        local clone = spawned:Clone()
        clone.Name = "EchoPreview_" .. toyName

        for _, p in ipairs(clone:GetDescendants()) do
            if p:IsA("BasePart") then
                p.Anchored       = true
                p.CanCollide     = false
                p.CanTouch       = false
                p.CanQuery       = false
                p.CastShadow     = false
            elseif p:IsA("Script") or p:IsA("LocalScript") then
                p:Destroy()
            elseif p:IsA("ParticleEmitter") or p:IsA("Trail") or p:IsA("Beam") then
                -- Turn off emitters that would produce motion / sparks in a static preview.
                pcall(function() p.Enabled = false end)
            end
        end

        local ok, pivotCF = pcall(function()
            return clone:GetPivot()
        end)

        clone.Parent = PreviewViewport

        -- Center at origin and orient nicely for the viewport.
        pcall(function()
            clone:PivotTo(CFrame.new(0, 0, 0))
        end)

        previewModel = clone

        -- Immediately destroy the server-side copy so it doesn't stay in your folder.
        if destroyRemote then
            pcall(function() destroyRemote:FireServer(spawned) end)
        end

        -- Auto-fit camera distance to the model's size.
        local centerCF, size = computePreviewPivot(clone)
        local maxExtent = math.max(size.X, size.Y, size.Z)
        previewDistance = math.max(maxExtent * 2.2, 5)

        -- Give the model a slight initial yaw so its features are visible.
        previewYaw = 30
        previewPitch = 15
    end

    -- Auto-spin + camera update
    task.spawn(function()
        while PreviewViewport.Parent do
            if previewAutoSpin and not previewDragging then
                previewYaw = previewYaw + 0.6
            end
            PreviewCamera.CFrame = computePreviewCFrame()
            RunService.RenderStepped:Wait()
        end
    end)

    -- Drag-to-rotate
    PreviewViewport.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            previewDragging = true
            previewLastMouse = Vector2.new(input.Position.X, input.Position.Y)
        end
    end)

    PreviewViewport.InputChanged:Connect(function(input)
        if not previewDragging then return end
        if input.UserInputType ~= Enum.UserInputType.MouseMovement
            and input.UserInputType ~= Enum.UserInputType.Touch then return end

        local pos = Vector2.new(input.Position.X, input.Position.Y)
        local delta = pos - previewLastMouse
        previewLastMouse = pos

        previewYaw   = previewYaw - delta.X * 0.5
        previewPitch = math.clamp(previewPitch + delta.Y * 0.5, -80, 80)
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            previewDragging = false
        end
    end)

    -- ── Target dropdown ───────────────────────────────────────
    local function GetExplosionPlayerList()
        local list = {}
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                table.insert(list, player.DisplayName .. " (@" .. player.Name .. ")")
            end
        end
        table.sort(list, function(a, b) return a:lower() < b:lower() end)
        return list
    end

    local function GetExplosionUsername(value)
        if type(value) ~= "string" then return nil end
        return value:match("@([%w_]+)") or value
    end

    local ExplosionTargetDropdown = TargetExplosionGroup:AddDropdown("EchoExplosionTarget", {
        Values = GetExplosionPlayerList(),
        Default = nil,
        Text = "Target Player",
        Multi = false,
        Searchable = true,
        AllowNull = true,
        Tooltip = "Choose the player to target with explosions.",
        Callback = function(Value)
            TargetExplosionState.Target = GetExplosionUsername(Value)
        end,
    })

    TargetExplosionGroup:AddButton({
        Text = "Refresh List",
        Func = function()
            pcall(function() ExplosionTargetDropdown:SetValues(GetExplosionPlayerList()) end)
        end,
    })

    TargetExplosionGroup:AddDivider()

    -- Explosion type dropdown
    TargetExplosionGroup:AddDropdown("EchoExplosionType", {
        Values = { "Missile", "Firework", "Void", "Balloon", "Small Present", "Big Present", "Snowball" },
        Default = "Missile",
        Text = "Explosion Type",
        Tooltip = "Choose the explosion toy. Drag the preview above to rotate it.",
        Callback = function(Value)
            if ExplosionTypes[Value] then
                if TargetExplosionState.Enabled then
                    DeleteExplosionBombs(ExplosionTypes[TargetExplosionState.Type] or "BombMissile")
                end
                TargetExplosionState.Type = Value
                loadPreviewModel(ExplosionTypes[Value])
            end
        end,
    })

    -- ── Bomb helpers ──────────────────────────────────────────
    local function GetExplosionTargets()
        local targets = {}
        if TargetExplosionState.Target then
            local player = Players:FindFirstChild(TargetExplosionState.Target)
            if player and player ~= LocalPlayer then
                table.insert(targets, player)
            end
        end
        return targets
    end

    local function GetExplosionFolder()
        return Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
    end

    local function GetExplosionBombs(toyName)
        local folder = GetExplosionFolder()
        local bombs = {}
        if not folder then return bombs end
        for _, toy in ipairs(folder:GetChildren()) do
            if toy.Name == toyName then
                table.insert(bombs, toy)
            end
        end
        return bombs
    end

    local function DeleteExplosionBombs(toyName)
        for _, bomb in ipairs(GetExplosionBombs(toyName)) do
            pcall(function()
                local menu = ReplicatedStorage:FindFirstChild("MenuToys")
                local destroyRemote = menu and menu:FindFirstChild("DestroyToy")
                if destroyRemote then destroyRemote:FireServer(bomb) end
            end)
        end
    end

    local function GetExplosionRoot()
        local character = LocalPlayer.Character
        return character and character:FindFirstChild("HumanoidRootPart")
    end

    local function SetExplosionOwnership(part)
        if not part then return end
        local root = GetExplosionRoot()
        local grabEvents = ReplicatedStorage:FindFirstChild("GrabEvents")
        local ownerRemote = grabEvents and grabEvents:FindFirstChild("SetNetworkOwner")
        if not root or not ownerRemote then return end
        pcall(function()
            ownerRemote:FireServer(part, CFrame.lookAt(root.Position, part.Position))
        end)
    end

    local function SpawnExplosionBomb(toyName)
        local root = GetExplosionRoot()
        local menu = ReplicatedStorage:FindFirstChild("MenuToys")
        local spawnRemote = menu and menu:FindFirstChild("SpawnToyRemoteFunction")
        if not root or not spawnRemote then return nil end

        local folder = GetExplosionFolder()
        local before = {}
        if folder then
            for _, child in ipairs(folder:GetChildren()) do before[child] = true end
        end

        pcall(function()
            spawnRemote:InvokeServer(toyName, root.CFrame * CFrame.new(0, 5, 0), Vector3.zero)
        end)

        for _ = 1, 10 do
            task.wait(0.005)
            folder = GetExplosionFolder()
            if folder then
                for _, child in ipairs(folder:GetChildren()) do
                    if child.Name == toyName and not before[child] then return child end
                end
            end
        end

        local bombs = GetExplosionBombs(toyName)
        return bombs[#bombs]
    end

    local function SetupExplosionBomb(bomb)
        if not bomb then return end
        local hitPartName = ExplosionSetupParts[bomb.Name]
        local hitPart = hitPartName and bomb:FindFirstChild(hitPartName)
        local primary = bomb.PrimaryPart or hitPart or bomb:FindFirstChildWhichIsA("BasePart", true)
        if not hitPart then hitPart = primary end
        if not hitPart or not primary then return end

        SetExplosionOwnership(hitPart)

        pcall(function()
            for _, child in ipairs(primary:GetChildren()) do
                if child:IsA("BodyVelocity") and child.Name == "EchoExplosionStable" then
                    child:Destroy()
                end
            end

            local bodyVelocity = Instance.new("BodyVelocity")
            bodyVelocity.Name = "EchoExplosionStable"
            bodyVelocity.Velocity = Vector3.zero
            bodyVelocity.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bodyVelocity.Parent = primary

            bomb:PivotTo(CFrame.new(math.random(-500, 500), 10000, math.random(-500, 500)))
        end)
    end

    -- Predicted-movement explosion. Aims at targetRoot.Position + Velocity/1.93
    local function ExplodeBombAtTarget(bomb, target)
        if not bomb or not target or not target.Character then return end
        local targetRoot = target.Character:FindFirstChild("HumanoidRootPart")
        local hitboxName = ExplosionHitboxes[bomb.Name]
        local hitbox = hitboxName and bomb:FindFirstChild(hitboxName)
        if not hitbox then hitbox = bomb:FindFirstChildWhichIsA("BasePart", true) end
        local bombEvents = ReplicatedStorage:FindFirstChild("BombEvents")
        local explodeRemote = bombEvents and bombEvents:FindFirstChild("BombExplode")
        if not targetRoot or not hitbox or not explodeRemote then return end

        local predicted = targetRoot.Position + targetRoot.AssemblyLinearVelocity / 1.93

        pcall(function()
            explodeRemote:FireServer(
                { Hitbox = hitbox, PositionPart = targetRoot },
                predicted
            )
        end)
    end

    local function RunTargetExplosionCycle()
        local toyName = ExplosionTypes[TargetExplosionState.Type] or "BombMissile"
        local targets = GetExplosionTargets()
        if #targets == 0 then return end

        DeleteExplosionBombs(toyName)
        task.wait(0.01)

        local bombs = {}
        for _ = 1, TargetExplosionState.Amount do
            if not TargetExplosionState.Enabled and TargetExplosionState.Thread then break end
            local bomb = SpawnExplosionBomb(toyName)
            if bomb then table.insert(bombs, bomb) end
        end

        task.wait(0.005)
        for _, bomb in ipairs(bombs) do
            if TargetExplosionState.Thread and not TargetExplosionState.Enabled then break end
            SetupExplosionBomb(bomb)
        end

        task.wait(0.005)
        for _, target in ipairs(targets) do
            if TargetExplosionState.Thread and not TargetExplosionState.Enabled then break end
            for _, bomb in ipairs(bombs) do
                if TargetExplosionState.Thread and not TargetExplosionState.Enabled then break end
                ExplodeBombAtTarget(bomb, target)
            end
        end

        task.wait(0.01)
        DeleteExplosionBombs(toyName)
    end

    local function StopTargetExplosion()
        TargetExplosionState.Enabled = false
        if TargetExplosionState.Thread then
            pcall(task.cancel, TargetExplosionState.Thread)
            TargetExplosionState.Thread = nil
        end
        DeleteExplosionBombs(ExplosionTypes[TargetExplosionState.Type] or "BombMissile")
    end

    local function StartTargetExplosion()
        if TargetExplosionState.Thread then
            pcall(task.cancel, TargetExplosionState.Thread)
        end
        TargetExplosionState.Enabled = true
        TargetExplosionState.Thread = task.spawn(function()
            while TargetExplosionState.Enabled do
                pcall(RunTargetExplosionCycle)
                task.wait(0.01)
            end
        end)
    end

    TargetExplosionGroup:AddToggle("EchoTargetExplosion", {
        Text = "Loop Explode",
        Default = false,
        Tooltip = "Continuously spawns explosion toys and detonates them on your selected target.",
        Callback = function(Value)
            if Value then StartTargetExplosion() else StopTargetExplosion() end
        end,
    })

    TargetExplosionGroup:AddSlider("EchoExplosionAmount", {
        Text = "Amount",
        Default = 3,
        Min = 1,
        Max = 10,
        Rounding = 0,
        Suffix = " bombs",
        Callback = function(Value)
            TargetExplosionState.Amount = math.clamp(math.floor(Value), 1, 10)
        end,
    })

    TargetExplosionGroup:AddButton({
        Text = "Explode Once",
        Tooltip = "Spawns the configured bombs and detonates them on the selected target once.",
        Func = function()
            if TargetExplosionState.Enabled then return end
            task.spawn(function()
                -- Fire one cycle even though the loop toggle is off
                TargetExplosionState.Enabled = true
                RunTargetExplosionCycle()
                TargetExplosionState.Enabled = false
            end)
        end,
    })

    -- Initial preview load
    loadPreviewModel("BombMissile")

    -- Refresh explosion target dropdown on player add/remove
    Players.PlayerAdded:Connect(function()
        task.wait(0.5)
        pcall(function() ExplosionTargetDropdown:SetValues(GetExplosionPlayerList()) end)
    end)
    Players.PlayerRemoving:Connect(function()
        task.wait(0.5)
        pcall(function() ExplosionTargetDropdown:SetValues(GetExplosionPlayerList()) end)
    end)

    -- ═══════════════════════════════════════════════════════════
    -- RIGHT GROUPBOX — "Train"
    -- ═══════════════════════════════════════════════════════════
    local TrainGroup = MiscTab:AddRightGroupbox("Train", "train-front")

    TrainGroup:AddButton({
        Text = "Bring Train",
        Tooltip = "Use vfly in IY while holding a burger to bring the train",
        Func = function()
            local rs = game:GetService("ReplicatedStorage")
            local plr = LocalPlayer
            local char = plr.Character or plr.CharacterAdded:Wait()
            local HRP = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChild("Humanoid")
            local inv = workspace:FindFirstChild(plr.Name .. "SpawnedInToys")

            local DestroyToy = rs:WaitForChild("MenuToys", 5) and rs.MenuToys:FindFirstChild("DestroyToy")
            local SpawnToy = rs:WaitForChild("MenuToys", 5) and rs.MenuToys:FindFirstChild("SpawnToyRemoteFunction")

            if not HRP or not hum or not inv then return end

            local function getplot()
                for i = 1, 5 do
                    local plot = workspace.Plots:FindFirstChild("Plot"..i)
                    local value = plot and plot:FindFirstChild("PlotSign") and plot.PlotSign:FindFirstChild("ThisPlotsOwners") and plot.PlotSign.ThisPlotsOwners:FindFirstChild("Value")
                    if plot and value and value.Value:find(plr.Name) then
                        return plot
                    end
                end
                return nil
            end

            local function spawntoy(toy, cf)
                if plr:FindFirstChild("CanSpawnToy") and not plr.CanSpawnToy.Value then
                    plr.CanSpawnToy.Changed:Wait()
                end

                local t
                local toyadded
                toyadded = inv.ChildAdded:Connect(function(c)
                    if c.Name == toy then
                        t = c
                        toyadded:Disconnect()
                    end
                end)

                task.spawn(function()
                    pcall(function()
                        SpawnToy:InvokeServer(toy, cf, Vector3.new(0, 0, 0))
                    end)
                end)

                local time = tick() + 1
                repeat task.wait() until t or tick() > time

                if t then
                    return t
                else
                    local plot = getplot()
                    if plot then
                        local plotItems = workspace:FindFirstChild("PlotItems")
                        if plotItems and plotItems:FindFirstChild(plot.Name) then
                            return plotItems[plot.Name]:FindFirstChild(toy) or plotItems[plot.Name]:WaitForChild(toy, 0.5)
                        end
                    end
                end
            end

            local function grab(obj)
                if obj and obj:FindFirstChild("HoldPart") and obj.HoldPart:FindFirstChild("HoldItemRemoteFunction") then
                    pcall(function()
                        obj.HoldPart.HoldItemRemoteFunction:InvokeServer(obj, char)
                    end)
                end
            end

            local pos = HRP.CFrame
            local burger = spawntoy("FoodHamburger", HRP.CFrame)

            if burger then
                repeat task.wait() until burger:FindFirstChild("HoldPart")

                local map = workspace:FindFirstChild("Map")
                local train = map and map:FindFirstChild("AlwaysHereTweenedObjects") and map.AlwaysHereTweenedObjects:FindFirstChild("Train")

                if train and train:FindFirstChild("Object") then
                    local trainObj = train.Object
                    local model = trainObj:FindFirstChild("ObjectModel")
                    local followPart = trainObj:FindFirstChild("FollowThisPart")

                    if model and model:FindFirstChild("Seat") then
                        model.Seat:Sit(hum)
                    end

                    if followPart then
                        if followPart:FindFirstChild("AlignPosition") then
                            followPart.AlignPosition.Enabled = false
                        end
                        if followPart:FindFirstChild("AlignOrientation") then
                            followPart.AlignOrientation.Enabled = false
                        end
                    end
                end

                task.wait(0.1)
                grab(burger)
                task.wait(0.1)

                pcall(function()
                    DestroyToy:FireServer(burger)
                end)

                HRP.CFrame = pos * CFrame.new(0, 5, 0)
            else
                Library:Notify({
                    Title = "Echo",
                    Description = "Failed to spawn Hamburger.",
                    Time = 3,
                })
            end
        end,
    })

    -- ═══════════════════════════════════════════════════════════
    -- CLEANUP
    -- ═══════════════════════════════════════════════════════════
    pcall(function()
        Library:OnUnload(function()
            stopCoconutPenis()
            stopCoconutBoobs()
            stopCoconutAss()
            stopCoconut67()
            stopCoconut67Animation()
            StopTargetExplosion()
            clearPreviewModel()
        end)
    end)
end
--endmisc tab


local State = {
    WalkspeedValue = 1,
    WalkspeedEnabled = false,
    JumpPowerValue = 24,
    FlightSpeed = 1,
    FOV = 70,
    SelectedLocation = "Spawn",
    SelectedPlayer = nil,
    ThirdPerson = false,
    UIKey = Enum.KeyCode.RightShift,
}

local ToggleAction = "Echo_UI_Toggle"

function GetMainGui()
    local KeybindFrame = Library.KeybindFrame
    if not KeybindFrame then
        return nil
    end
    local Gui = KeybindFrame.Parent
    if Gui and Gui:IsA("ScreenGui") then
        return Gui
    end
    return nil
end

function IsMenuOpen()
    local Gui = GetMainGui()
    return Gui ~= nil and Gui.Enabled
end

function ApplyCursor()
    pcall(function()
        if IsMenuOpen() then
            UserInputService.MouseIconEnabled = true
            UserInputService.MouseBehavior = Enum.MouseBehavior.Default
        elseif LocalPlayer.CameraMode == Enum.CameraMode.LockFirstPerson then
            UserInputService.MouseIconEnabled = false
            UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
        else
            UserInputService.MouseIconEnabled = true
            UserInputService.MouseBehavior = Enum.MouseBehavior.Default
        end
    end)
end

function ToggleUI()
    local Gui = GetMainGui()
    if not Gui then
        return
    end
    Gui.Enabled = not Gui.Enabled
    ApplyCursor()
end

function BindUIKey(Key)
    if Key then
        State.UIKey = Key
    end
    ContextActionService:UnbindAction(ToggleAction)
    ContextActionService:BindAction(ToggleAction, function(_, InputState)
        if InputState == Enum.UserInputState.Begin then
            ToggleUI()
        end
        return Enum.ContextActionResult.Sink
    end, false, State.UIKey)
end

BindUIKey()
ApplyCursor()

function GetGreeting()
    local Hour = tonumber(os.date("%H"))
    if Hour >= 5 and Hour < 12 then
        return "<b><font color='#6FCF97'>Morning ☀️</font></b>, Thank you for using Echo"
    elseif Hour >= 12 and Hour < 17 then
        return "<b><font color='#F2C94C'>Afternoon 🌤️</font></b>, Thank you for using Echo"
    elseif Hour >= 17 and Hour < 21 then
        return "<b><font color='#F2994A'>Evening 🌆</font></b>, Thank you for using Echo"
    else
        return "<b><font color='#9B8AFB'>Night 🌙</font></b>, Thank you for using Echo"
    end
end

--main tab
local ProfileGroup = Tabs.Main:AddLeftGroupbox("Profile", "user")
local MainRightTabbox = Tabs.Main:AddRightTabbox("Info")
local InfoGroup = MainRightTabbox:AddTab("Info", "activity")
local StatsGroup = MainRightTabbox:AddTab("Stats", "clock")

-- CREDITS
local CreditsTab = MainRightTabbox:AddTab("Helpers", "users")

CreditsTab:AddLabel("Owner : <font color='#DC143C'>zyvr1</font> | @7egend7", true)
CreditsTab:AddLabel("Dev: <font color='#ff69b4'>lxoc</font> | @mikani_682", true)
CreditsTab:AddLabel("Dev: <font color='#ff69b4'>lxoc</font> | @rosey131",true)
local Thumbnail
pcall(function()
    Thumbnail = Players:GetUserThumbnailAsync(LocalPlayer.UserId, Enum.ThumbnailType.AvatarBust, Enum.ThumbnailSize.Size420x420)
end)

if Thumbnail then
    ProfileGroup:AddImage("Avatar", {
        Image = Thumbnail,
        Height = 130,
        ScaleType = Enum.ScaleType.Fit,
    })
end

ProfileGroup:AddLabel("<b>" .. LocalPlayer.DisplayName .. "</b>", true)
ProfileGroup:AddLabel("@" .. LocalPlayer.Name, true)
local GreetingLabel = ProfileGroup:AddLabel(GetGreeting(), true)
pcall(function()
    local GreetingGradient = Instance.new("UIGradient")
    GreetingGradient.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(0.35, 0),
        NumberSequenceKeypoint.new(0.65, 0),
        NumberSequenceKeypoint.new(1, 1),
    })
    GreetingGradient.Offset = Vector2.new(-1, 0)
    GreetingGradient.Parent = GreetingLabel
    task.spawn(function()
        while GreetingLabel and GreetingLabel.Parent and not Library.Unloaded do
            GreetingGradient.Offset = Vector2.new(-1, 0)
            local tween = TweenService:Create(GreetingGradient, TweenInfo.new(1.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Offset = Vector2.new(1, 0)})
            tween:Play()
            task.wait(1.8)
        end
    end)
end)
ProfileGroup:AddDivider()

ProfileGroup:AddButton({
    Text = "Discord",
    Func = function()
        local Copied = false
        pcall(function()
            if setclipboard then
                setclipboard(Discord)
                Copied = true
            elseif toclipboard then
                toclipboard(Discord)
                Copied = true
            elseif EchoEnv.syn and EchoEnv.syn.write_clipboard then
                EchoEnv.syn.write_clipboard(Discord)
                Copied = true
            end
        end)
        if Copied then
            _G.__EchoAnimeLibrary:Notify({Description = "Discord invite copied.", Time = 3})
        else
            Library:Notify({Description = "Clipboard is not supported.", Time = 3})
        end
    end,
})

ProfileGroup:AddButton({
    Text = "Server Hop",
    Func = function()
        Library:Notify({Description = "Searching for a new server...", Time = 2})
        task.spawn(function()
            local ok, result = pcall(function()
                local body = EchoHttpGet("https://games.roblox.com/v1/games/" .. tostring(game.PlaceId) .. "/servers/Public?sortOrder=Asc&limit=100")
                return game:GetService("HttpService"):JSONDecode(body)
            end)

            if not ok or type(result) ~= "table" or type(result.data) ~= "table" then
                Library:Notify({Description = "Server hop failed.", Time = 3})
                return
            end

            local candidates = {}
            for _, server in ipairs(result.data) do
                if server.id and server.id ~= game.JobId and tonumber(server.playing) and tonumber(server.maxPlayers) and server.playing < server.maxPlayers then
                    table.insert(candidates, server.id)
                end
            end

            if #candidates == 0 then
                Library:Notify({Description = "No available server found.", Time = 3})
                return
            end

            local targetJob = candidates[math.random(1, #candidates)]
            Library:Notify({Description = "Joining a new server...", Time = 2})
            task.wait(0.25)
            pcall(function()
                TeleportService:TeleportToPlaceInstance(game.PlaceId, targetJob, LocalPlayer)
            end)
        end)
    end,
})

ProfileGroup:AddButton({
    Text = "Unload",
    Func = function()
        pcall(function()
            Window:AddDialog("EchoUnloadConfirm", {
                Title = "Are you sure?",
                Description = "Are you sure you want to unload Echo?",
                AutoDismiss = false,
                OutsideClickDismiss = true,
                FooterButtons = {
                    Cancel = {
                        Title = "Cancel",
                        Variant = "Secondary",
                        Order = 1,
                        Callback = function(Dialog)
                            Dialog:Dismiss()
                        end,
                    },
                    Unload = {
                        Title = "Unload",
                        Variant = "Destructive",
                        Order = 2,
                        Callback = function(Dialog)
                            Dialog:Dismiss()
                            task.defer(function()
                                Library:Unload()
                            end)
                        end,
                    },
                },
            })
        end)
    end,
})

ProfileGroup:AddButton({
    Text = "Rejoin Server",
    Func = function()
        Library:Notify({Description = "Rejoining server...", Time = 2})
        task.wait(0.35)
        pcall(function()
            TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LocalPlayer)
        end)
    end,
})

InfoGroup:AddLabel("Game", true)
InfoGroup:AddLabel("Fling Things and People", false)
InfoGroup:AddDivider()
InfoGroup:AddLabel("Players", true)
InfoGroup:AddLabel(tostring(#Players:GetPlayers()) .. " / " .. tostring(Players.MaxPlayers), false)
InfoGroup:AddDivider()

local KicksDealt = 0
InfoGroup:AddLabel("Kicks", true)
local KickCountLabel = InfoGroup:AddLabel("Kicks Dealt: 0", false)

function AddKick()
    KicksDealt = KicksDealt + 1
    if KickCountLabel then
        pcall(function()
            KickCountLabel:SetText("Kicks Dealt: " .. tostring(KicksDealt))
        end)
    end
end

InfoGroup:AddDivider()

InfoGroup:AddLabel("Executor", true)
local ExecutorName = "Unknown"

pcall(function()
    if identifyexecutor then
        local Name = identifyexecutor()
        if Name and Name ~= "" then
            ExecutorName = Name
        end
    elseif getexecutorname then
        local Name = getexecutorname()
        if Name and Name ~= "" then
            ExecutorName = Name
        end
    elseif EchoEnv.fluxus then
        ExecutorName = "Fluxus"
    elseif EchoEnv.syn then
        ExecutorName = "Synapse"
    elseif EchoEnv.KRNL_LOADED then
        ExecutorName = "KRNL"
    elseif EchoEnv.is_sirhurt_closure then
        ExecutorName = "SirHurt"
    end
end)

InfoGroup:AddLabel(ExecutorName, false)
InfoGroup:AddLabel("Full Support", true)

StatsGroup:AddLabel("Script Version", true)
StatsGroup:AddLabel("<b>Echo " .. Version .. "</b>", true)
StatsGroup:AddDivider()

local SessionTimeLabel = StatsGroup:AddLabel("Session Time: 00:00:00", true)
local StartTime = os.clock()

task.spawn(function()
    while not Library.Unloaded do
        local Elapsed = math.floor(os.clock() - StartTime)
        local Hours = math.floor(Elapsed / 3600)
        local Minutes = math.floor((Elapsed % 3600) / 60)
        local Seconds = Elapsed % 60
        pcall(function()
            SessionTimeLabel:SetText(string.format("Session Time: %02d:%02d:%02d", Hours, Minutes, Seconds))
        end)
        task.wait(1)
    end
end)

--endmain tab

--player tab
local ValuesGroup = Tabs.Player:AddLeftGroupbox("Values", "sliders-horizontal")
local CharacterGroup = Tabs.Player:AddRightGroupbox("Character", "user")

local WalkspeedConnection = nil
local JumpPowerConnection = nil
local FlyVelocity = nil
local FlyGyro = nil
local FlyConnection = nil
local FlyInputs = {}
local JerkTrack = nil
local NoclipConnection = nil
local InfiniteJumpConnection = nil

local OriginalCameraMode = LocalPlayer.CameraMode
local OriginalMinZoom = LocalPlayer.CameraMinZoomDistance
local OriginalMaxZoom = LocalPlayer.CameraMaxZoomDistance

local FlyVelocity = nil
local FlyGyro = nil
local FlyConnection = nil

function StopFlight()
    if FlyConnection then
        FlyConnection:Disconnect()
        FlyConnection = nil
    end
    if FlyVelocity then
        FlyVelocity:Destroy()
        FlyVelocity = nil
    end
    if FlyGyro then
        FlyGyro:Destroy()
        FlyGyro = nil
    end
    local char = LocalPlayer.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        local root = char:FindFirstChild("HumanoidRootPart")
        if hum then
            hum.PlatformStand = false
            hum:ChangeState(Enum.HumanoidStateType.GettingUp)
        end
        if root then
            root.AssemblyLinearVelocity = Vector3.zero
            root.AssemblyAngularVelocity = Vector3.zero
        end
    end
end

function StartFlight()
    StopFlight()
    if not Toggles.FlightToggle or not Toggles.FlightToggle.Value then return end

    FlyConnection = RunService.RenderStepped:Connect(function()
        local char = LocalPlayer.Character
        if not char then return end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local root = char:FindFirstChild("HumanoidRootPart")
        if not hum or not root then return end

        local controlPart = (hum.Sit and hum.SeatPart and hum.SeatPart.AssemblyRootPart) or root

        if not FlyVelocity or FlyVelocity.Parent ~= controlPart then
            if FlyVelocity then FlyVelocity:Destroy() end
            FlyVelocity = Instance.new("BodyVelocity")
            FlyVelocity.MaxForce = Vector3.new(1e9, 1e9, 1e9)
            FlyVelocity.Parent = controlPart
        end

        if not FlyGyro or FlyGyro.Parent ~= controlPart then
            if FlyGyro then FlyGyro:Destroy() end
            FlyGyro = Instance.new("BodyGyro")
            FlyGyro.MaxTorque = Vector3.new(1e9, 1e9, 1e9)
            FlyGyro.D = 100
            FlyGyro.Parent = controlPart
        end

        local move = Vector3.zero
        if UserInputService:IsKeyDown(Enum.KeyCode.W) then move = move + Vector3.new(0, 0, -1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.S) then move = move + Vector3.new(0, 0, 1) end
        if UserInputService:IsKeyDown(Enum.KeyCode.A) then move = move + Vector3.new(-1, 0, 0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.D) then move = move + Vector3.new(1, 0, 0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then move = move + Vector3.new(0, -1, 0) end

        local camCF = Workspace.CurrentCamera.CFrame
        local speed = State.FlightSpeed * 50

        if move.Magnitude > 0 then
            FlyVelocity.Velocity = (camCF.LookVector * -move.Z + camCF.RightVector * move.X + Vector3.new(0, move.Y, 0)) * speed
        else
            FlyVelocity.Velocity = Vector3.zero
        end

        FlyGyro.CFrame = CFrame.new(controlPart.Position, controlPart.Position + camCF.LookVector)
        hum.PlatformStand = not hum.Sit
    end)
end

ValuesGroup:AddSlider("WalkSpeedValue", {
    Text = "Walk Speed",
    Default = 1,
    Min = 1,
    Max = 15,
    Rounding = 0,
    Callback = function(Value)
        State.WalkspeedValue = Value
    end,
})

CharacterGroup:AddToggle("WalkspeedToggle", {
    Text = "Walkspeed",
    Default = false,
    Callback = function(Value)
        State.WalkspeedEnabled = Value

        if WalkspeedConnection then
            WalkspeedConnection:Disconnect()
            WalkspeedConnection = nil
        end

        if not Value then
            return
        end

        WalkspeedConnection = RunService.Stepped:Connect(function()
            if not State.WalkspeedEnabled then
                return
            end

            local Character = LocalPlayer.Character
            local Root = Character and Character:FindFirstChild("HumanoidRootPart")
            local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
            local Speed = tonumber(State.WalkspeedValue)

            if Root and Humanoid and Speed then
                local MoveDirection = Humanoid.MoveDirection
                if MoveDirection.Magnitude > 0 then
                    Root.CFrame = Root.CFrame + MoveDirection * ((16 * Speed) / 10)
                end
            end
        end)
    end,
})

Toggles.WalkspeedToggle:AddKeyPicker("WalkspeedKey", {
    Text = "Keybind",
    Default = "None",
    Mode = "Toggle",
    SyncToggleState = true,
})

ValuesGroup:AddSlider("JumpPowerValue", {
    Text = "Jump Power",
    Default = 1,
    Min = 1,
    Max = 150,
    Rounding = 0,
    Callback = function(Value)
        State.JumpPowerValue = Value
        local Toggle = Toggles.JumpPowerToggle
        if Toggle and Toggle.Value then
            local Character = LocalPlayer.Character
            local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
            if Humanoid then
                Humanoid.JumpPower = Value
            end
        end
    end,
})

CharacterGroup:AddToggle("JumpPowerToggle", {
    Text = "Jump Power",
    Default = false,
    Callback = function(Value)
        local Character = LocalPlayer.Character
        local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
        if not Humanoid then return end

        if Value then
            Humanoid.JumpPower = State.JumpPowerValue or 24
            if JumpPowerConnection then JumpPowerConnection:Disconnect() end
            JumpPowerConnection = Humanoid:GetPropertyChangedSignal("JumpPower"):Connect(function()
                local Toggle = Toggles.JumpPowerToggle
                if Toggle and Toggle.Value and Humanoid.Parent then
                    local Power = State.JumpPowerValue or 24
                    if Humanoid.JumpPower ~= Power then
                        Humanoid.JumpPower = Power
                    end
                end
            end)
        else
            if JumpPowerConnection then
                JumpPowerConnection:Disconnect()
                JumpPowerConnection = nil
            end
            Humanoid.JumpPower = 50
        end
    end,
})

Toggles.JumpPowerToggle:AddKeyPicker("JumpPowerKey", {
    Text = "Keybind",
    Default = "None",
    Mode = "Toggle",
    SyncToggleState = true,
})

ValuesGroup:AddSlider("FlightSpeedValue", {
    Text = "Flight Speed",
    Default = 1,
    Min = 1,
    Max = 20,
    Rounding = 0,
    Callback = function(Value)
        State.FlightSpeed = Value
    end,
})

CharacterGroup:AddToggle("FlightToggle", {
    Text = "Flight",
    Default = false,
    Callback = function(Value)
        if Value then
            StartFlight()
        else
            StopFlight()
        end
    end,
})

Toggles.FlightToggle:AddKeyPicker("FlightKey", {
    Text = "Keybind",
    Default = "None",
    Mode = "Toggle",
    SyncToggleState = true,
})

function SetNoclip(Value)
    if NoclipConnection then
        NoclipConnection:Disconnect()
        NoclipConnection = nil
    end

    if not Value then
        local Character = LocalPlayer.Character
        if Character then
            for _, Part in ipairs(Character:GetDescendants()) do
                if Part:IsA("BasePart") then
                    Part.CanCollide = true
                end
            end
        end
        return
    end

    NoclipConnection = RunService.Stepped:Connect(function()
        local Character = LocalPlayer.Character
        if not Character then return end
        for _, Part in ipairs(Character:GetDescendants()) do
            if Part:IsA("BasePart") then
                Part.CanCollide = false
            end
        end
    end)
end

CharacterGroup:AddToggle("NoclipToggle", {
    Text = "Noclip",
    Default = false,
    Callback = function(Value)
        SetNoclip(Value)
    end,
})

function SetWaterWalk(Value)
    local Ocean = Workspace:FindFirstChild("Map")
    if Ocean then Ocean = Ocean:FindFirstChild("AlwaysHereTweenedObjects") end
    if Ocean then Ocean = Ocean:FindFirstChild("Ocean") end
    if Ocean then Ocean = Ocean:FindFirstChild("Object") end
    if Ocean then Ocean = Ocean:FindFirstChild("ObjectModel") end
    if not Ocean then return end

    for _, Part in ipairs(Ocean:GetChildren()) do
        if Part:IsA("BasePart") and Part.Name == "Ocean" then
            Part.CanCollide = Value
        end
    end
end

CharacterGroup:AddToggle("WaterWalkToggle", {
    Text = "Water Walk",
    Default = false,
    Callback = function(Value)
        SetWaterWalk(Value)
    end,
})

CharacterGroup:AddToggle("InfiniteJumpToggle", {
    Text = "Inf Jump",
    Default = false,
    Callback = function() end,
})

InfiniteJumpConnection = UserInputService.JumpRequest:Connect(function()
    local Toggle = Toggles.InfiniteJumpToggle
    if not Toggle or not Toggle.Value then return end
    local Character = LocalPlayer.Character
    local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
    if Humanoid then
        Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
    end
end)

local JerkAnimation = "rbxassetid://168268306"

function StopJerk()
    if JerkTrack then
        pcall(function()
            JerkTrack:Stop()
            JerkTrack:Destroy()
        end)
        JerkTrack = nil
    end
end

function StartJerk()
    StopJerk()
    local Character = LocalPlayer.Character
    local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
    if not Humanoid then return end

    local Animator = Humanoid:FindFirstChildOfClass("Animator")
    if not Animator then
        Animator = Instance.new("Animator")
        Animator.Parent = Humanoid
    end

    local Animation = Instance.new("Animation")
    Animation.AnimationId = JerkAnimation

    local Success, Track = pcall(function()
        return Animator:LoadAnimation(Animation)
    end)

    Animation:Destroy()

    if not Success or not Track then return end

    JerkTrack = Track
    JerkTrack.Priority = Enum.AnimationPriority.Action
    JerkTrack:Play()

    task.spawn(function()
        while true do
            local Toggle = Toggles.JerkToggle
            if not Toggle or not Toggle.Value or JerkTrack ~= Track then break end
            task.wait(0.1)
            if Track.IsPlaying then
                pcall(function()
                    Track.TimePosition = 0.3
                end)
            end
        end
    end)
end

CharacterGroup:AddToggle("JerkToggle", {
    Text = "Jerk Off",
    Default = false,
    Callback = function(Value)
        if Value then
            StartJerk()
        else
            StopJerk()
        end
    end,
})

Toggles.JerkToggle:AddKeyPicker("JerkKey", {
    Text = "Keybind",
    Default = "None",
    Mode = "Toggle",
    SyncToggleState = true,
})

ValuesGroup:AddSlider("FOVValue", {
    Text = "FOV",
    Default = 70,
    Min = 1,
    Max = 120,
    Rounding = 0,
    Callback = function(Value)
        State.FOV = Value

        pcall(function()
            local Camera = Workspace.CurrentCamera
            if Camera then
                Camera.FieldOfView = Value
            end
        end)
    end,
})

function SetThirdPerson(Value)
    State.ThirdPerson = Value

    if Value then
        pcall(function()
            LocalPlayer.CameraMode = Enum.CameraMode.Classic
            LocalPlayer.CameraMinZoomDistance = 6
            LocalPlayer.CameraMaxZoomDistance = 100000
        end)
    else
        pcall(function()
            LocalPlayer.CameraMode = OriginalCameraMode
            LocalPlayer.CameraMinZoomDistance = OriginalMinZoom
            LocalPlayer.CameraMaxZoomDistance = OriginalMaxZoom
        end)
    end

    pcall(function()
        local Camera = Workspace.CurrentCamera
        if Camera then
            Camera.FieldOfView = State.FOV or 70
        end
    end)
end

CharacterGroup:AddToggle("ThirdPersonToggle", {
    Text = "Third Person",
    Default = false,
    Callback = function(Value)
        SetThirdPerson(Value)
    end,
})

Toggles.ThirdPersonToggle:AddKeyPicker("ThirdPersonKey", {
    Text = "Keybind",
    Default = "None",
    Mode = "Toggle",
    SyncToggleState = true,
})

local function EchoClearEntireInventory()
    local folderName = LocalPlayer.Name .. "SpawnedInToys"
    local toyFolder = Workspace:FindFirstChild(folderName)

    if not toyFolder then
        pcall(function()
            pcall(function()
                _G.__EchoNotificationSound.SoundId = "rbxassetid://97643101798871"
                _G.__EchoNotificationSound:Stop()
                _G.__EchoNotificationSound.TimePosition = 0
                _G.__EchoNotificationSound:Play()
            end)
            Library:Notify({
                Description = "Inventory Cleared",
                Time = 3,
                Image = "rbxassetid://" .. tostring(Logo),
            })
        end)
        return
    end

    for _, toy in ipairs(toyFolder:GetChildren()) do
        if toy.Name ~= "ToyNumber"
            and toy.Name ~= "완전한 사람"
            and not Players:GetPlayerFromCharacter(toy) then
            pcall(function()
                ReplicatedStorage:WaitForChild("MenuToys"):WaitForChild("DestroyToy"):FireServer(toy)
            end)
        end
    end

    pcall(function()
        pcall(function()
            _G.__EchoNotificationSound.SoundId = "rbxassetid://97643101798871"
            _G.__EchoNotificationSound:Stop()
            _G.__EchoNotificationSound.TimePosition = 0
            _G.__EchoNotificationSound:Play()
        end)
        Library:Notify({
            Description = "Inventory Cleared",
            Time = 3,
            Image = "rbxassetid://" .. tostring(Logo),
        })
    end)
end

CharacterGroup:AddButton("Clear Entire Inventory", function()
    EchoClearEntireInventory()
end)

local TeleportGroup = Tabs.Player:AddLeftGroupbox("Teleporting", "map-pin")

local Locations = {
    "Spawn", "SpawnCave", "GreenHouse", "PinkHouse", "Barn",
    "BlueHouse", "ChineseHouse", "PurpleHouse", "Factory",
    "OtherGreenHouse", "BigCave", "TrainCave", "IslandCave",
    "ChineseRoof", "UfoCave", "Prison", "GoodPrison",
    "RuhubsDogAhhPrison", "ExtremelyGoodPrison", "BlueHouseSlot",
    "SpawnSlot", "HauntedSlot", "RandomSlot", "BeachSlot",
}

local LocationCFrames = {
    Spawn = CFrame.new(0, -7.35, 0),
    SpawnCave = CFrame.new(-90, 14.6, -314.3),
    GreenHouse = CFrame.new(-538, -7, 74),
    PinkHouse = CFrame.new(-478, -7, -147),
    Barn = CFrame.new(-228, 82, -318),
    BlueHouse = CFrame.new(496, 83, -350),
    ChineseHouse = CFrame.new(542, 123, -93),
    PurpleHouse = CFrame.new(270, -7, 448),
    Factory = CFrame.new(134, 347, 352),
    OtherGreenHouse = CFrame.new(-359, 98, 357),
    BigCave = CFrame.new(-245, 80, 485),
    TrainCave = CFrame.new(536.6, 87.5, -169.5),
    IslandCave = CFrame.new(75.8, 323, 368.5),
    ChineseRoof = CFrame.new(592, 153, -100),
    UfoCave = CFrame.new(29.6, 10.5, -225.8),
    Prison = CFrame.new(195, -7, -561),
    GoodPrison = CFrame.new(569.6, -7, 176.3),
    RuhubsDogAhhPrison = CFrame.new(564, 82.5, 210),
    ExtremelyGoodPrison = CFrame.new(525, 76, 56),
    BlueHouseSlot = CFrame.new(562.2, 85.38, -212.56),
    SpawnSlot = CFrame.new(51.75, -5.3, -121.64),
    HauntedSlot = CFrame.new(164.57, -5.43, 530.97),
    RandomSlot = CFrame.new(-211.65, 85.7, 426.72),
    BeachSlot = CFrame.new(-546.97, -5.3, -41.09),
}

TeleportGroup:AddDropdown("LocationDropdown", {
    Text = "Location",
    Values = Locations,
    Default = "Spawn",
    Callback = function(Value)
        State.SelectedLocation = Value
    end,
})

TeleportGroup:AddToggle("LoopTeleport", {
    Text = "Loop TP to Location",
    Default = false,
    Callback = function(Value)
        if not Value then return end
        task.spawn(function()
            while true do
                local Toggle = Toggles.LoopTeleport
                if not Toggle or not Toggle.Value then break end
                local Target = LocationCFrames[State.SelectedLocation]
                if Target then
                    local Character = LocalPlayer.Character
                    local Root = Character and Character:FindFirstChild("HumanoidRootPart")
                    if Root then
                        Root.CFrame = Target
                        Root.AssemblyLinearVelocity = Vector3.zero
                        Root.AssemblyAngularVelocity = Vector3.zero
                    end
                end
                task.wait(0.1)
            end
        end)
    end,
})

TeleportGroup:AddButton({
    Text = "TP to Location",
    Func = function()
        local Target = LocationCFrames[State.SelectedLocation]
        if not Target then return end
        local Character = LocalPlayer.Character
        local Root = Character and Character:FindFirstChild("HumanoidRootPart")
        if Root then
            Root.CFrame = Target
            Root.AssemblyLinearVelocity = Vector3.zero
            Root.AssemblyAngularVelocity = Vector3.zero
        end
    end,
})

local PlayersGroup = Tabs.Player:AddRightGroupbox("Players", "users")

local function GetPlayerList()
    local List = {}
    for _, Player in ipairs(Players:GetPlayers()) do
        if Player ~= LocalPlayer then
            table.insert(List, Player.DisplayName .. " (@" .. Player.Name .. ")")
        end
    end
    table.sort(List)
    return List
end

PlayersGroup:AddDropdown("SelectPlayer", {
    Text = "Select Player",
    Values = GetPlayerList(),
    Default = nil,
    Callback = function(Value)
        local Username = Value and Value:match("@([%w_]+)") or Value
        State.SelectedPlayer = Username and Players:FindFirstChild(Username) or nil
    end,
})

function StopSpectating()
    local Character = LocalPlayer.Character
    local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
    local Camera = Workspace.CurrentCamera
    if Humanoid and Camera then
        Camera.CameraSubject = Humanoid
        Camera.CameraType = Enum.CameraType.Custom
    end
end

PlayersGroup:AddToggle("SpectatePlayer", {
    Text = "Spectate Player",
    Default = false,
    Callback = function(Value)
        local Target = State.SelectedPlayer
        if Value and Target then
            local Character = Target.Character
            local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
            local Camera = Workspace.CurrentCamera
            if Humanoid and Camera then
                Camera.CameraSubject = Humanoid
                Camera.CameraType = Enum.CameraType.Follow
            end
        else
            StopSpectating()
        end
    end,
})

PlayersGroup:AddToggle("LoopPlayer", {
    Text = "Loop TP to Player",
    Default = false,
    Callback = function(Value)
        if not Value then return end
        task.spawn(function()
            while true do
                local Toggle = Toggles.LoopPlayer
                if not Toggle or not Toggle.Value then break end

                local Target = State.SelectedPlayer
                local TargetCharacter = Target and Target.Character
                local TargetRoot = TargetCharacter and TargetCharacter:FindFirstChild("HumanoidRootPart")
                local Character = LocalPlayer.Character
                local Root = Character and Character:FindFirstChild("HumanoidRootPart")

                if TargetRoot and Root then
                    -- Smoothly follow the target instead of snapping every 0.1s.
                    local Goal = TargetRoot.CFrame * CFrame.new(0, 0, 3)
                    Root.CFrame = Root.CFrame:Lerp(Goal, 0.22)
                    Root.AssemblyLinearVelocity = Vector3.zero
                    Root.AssemblyAngularVelocity = Vector3.zero
                end

                task.wait()
            end
        end)
    end,
})

Players.PlayerAdded:Connect(function(Player)
    task.wait(0.5)
    pcall(function()
        if Options.SelectPlayer then
            Options.SelectPlayer:SetValues(GetPlayerList())
        end
    end)
end)

Players.PlayerRemoving:Connect(function(Player)
    if State.SelectedPlayer == Player then
        State.SelectedPlayer = nil
    end
    task.wait(0.5)
    pcall(function()
        if Options.SelectPlayer then
            Options.SelectPlayer:SetValues(GetPlayerList())
        end
    end)
end)

LocalPlayer.CharacterAdded:Connect(function(Character)
    task.wait(0.5)
    local Humanoid = Character:FindFirstChildOfClass("Humanoid")
    if not Humanoid then return end

    if Toggles.WalkspeedToggle and Toggles.WalkspeedToggle.Value then
        State.WalkspeedEnabled = true
        if WalkspeedConnection then
            WalkspeedConnection:Disconnect()
            WalkspeedConnection = nil
        end
        WalkspeedConnection = RunService.Stepped:Connect(function()
            if not State.WalkspeedEnabled then return end
            local CurrentCharacter = LocalPlayer.Character
            local Root = CurrentCharacter and CurrentCharacter:FindFirstChild("HumanoidRootPart")
            local CurrentHumanoid = CurrentCharacter and CurrentCharacter:FindFirstChildOfClass("Humanoid")
            local Speed = tonumber(State.WalkspeedValue)
            if Root and CurrentHumanoid and Speed then
                local MoveDirection = CurrentHumanoid.MoveDirection
                if MoveDirection.Magnitude > 0 then
                    Root.CFrame = Root.CFrame + MoveDirection * ((16 * Speed) / 10)
                end
            end
        end)
    end

    if Toggles.JumpPowerToggle
        and Toggles.JumpPowerToggle.Value then

        Humanoid.JumpPower =
            State.JumpPowerValue or 24
    end

    if Toggles.NoclipToggle and Toggles.NoclipToggle.Value then
        SetNoclip(true)
    end

    if Toggles.JerkToggle and Toggles.JerkToggle.Value then
        task.wait(0.25)
        StartJerk()
    end

    if Toggles.FlightToggle
        and Toggles.FlightToggle.Value then

        task.wait(0.25)

        StartFlight()
    end
end)

--endplayer tab
-- ============================================================
-- ANIMATIONS (Player tab — left side, below Teleporting)
-- ============================================================
do
    local AnimationsGroup = Tabs.Player:AddLeftGroupbox("Animations", "person-standing")

    local AnimPlayers = game:GetService("Players")
    local AnimPlayer  = AnimPlayers.LocalPlayer

    local animEnabled      = false
    local currentTrack     = nil
    local selectedAnimName = "Phantom Rush"
    local animSpeed        = 1

    local Animations = {
        ["Phantom Rush"]        = "rbxassetid://35154961",
        ["Ghostly Head"]        = "rbxassetid://121572214",
        ["Shadow Crouch"]       = "rbxassetid://182724289",
        ["Crawl of the Damned"] = "rbxassetid://282574440",
        ["Predator Walk"]       = "rbxassetid://204328711",
        ["Skybound Jacks"]      = "rbxassetid://429681631",
        ["Eternal Head"]        = "rbxassetid://35154961",
        ["Titan Leap"]          = "rbxassetid://184574340",
        ["Soul Faint"]          = "rbxassetid://181526230",
        ["Ground Collapse"]     = "rbxassetid://181525546",
        ["Void Faint"]          = "rbxassetid://181525546",
        ["Ascension"]           = "rbxassetid://313762630",
        ["Astral Sit"]          = "rbxassetid://179224234",
        ["Warped Motion"]       = "rbxassetid://215384594",
        ["Mirror Illusion"]     = "rbxassetid://215384594",
        ["Glitch Ascension"]    = "rbxassetid://313762630",
        ["Doom Punch"]          = "rbxassetid://204062532",
        ["Royal Bow"]           = "rbxassetid://204292303",
        ["Blade Impact"]        = "rbxassetid://204295235",
        ["Eternal Slam"]        = "rbxassetid://204295235",
        ["Chaos Insanity"]      = "rbxassetid://184574340",
        ["Meteor Punch"]        = "rbxassetid://126753849",
        ["Wraith Swing"]        = "rbxassetid://218504594",
        ["Cyclone Arms"]        = "rbxassetid://259438880",
        ["Barrel Storm"]        = "rbxassetid://136801964",
        ["Panic State"]         = "rbxassetid://180612465",
        ["Madness"]             = "rbxassetid://33796059",
        ["Arm Seeker"]          = "rbxassetid://33169583",
        ["Blade Slash"]         = "rbxassetid://35978879",
        ["Chaos Arms"]          = "rbxassetid://27432691",
        ["Victory Dab"]         = "rbxassetid://183412246",
        ["Vortex Spin"]         = "rbxassetid://188632011",
        ["Phantom Dance"]       = "rbxassetid://429703734",
        ["Spin of Kings"]       = "rbxassetid://429730430",
        ["Lunar Dance"]         = "rbxassetid://45834924",
        ["Whirl of Fate"]       = "rbxassetid://186934910",
        ["Thriller Night"]      = "rbxassetid://27789359",
        ["Mecha Mode"]          = "rbxassetid://30196114",
        ["Silent Shuffle"]      = "rbxassetid://248263260",
        ["Smooth Groove"]       = "rbxassetid://33796059",
        ["Club Fever"]          = "rbxassetid://28488254",
        ["Sky Bounce"]          = "rbxassetid://52155728",
        ["Wild Frenzy"]         = "rbxassetid://248263260",
        ["Final Fall"]          = "rbxassetid://35154961",
        ["Undead Stride"]       = "rbxassetid://33796059",
    }

    local AnimationList = {
        "Phantom Rush",
        "Ghostly Head",
        "Shadow Crouch",
        "Crawl of the Damned",
        "Predator Walk",
        "Skybound Jacks",
        "Eternal Head",
        "Titan Leap",
        "Soul Faint",
        "Ground Collapse",
        "Void Faint",
        "Ascension",
        "Astral Sit",
        "Warped Motion",
        "Mirror Illusion",
        "Glitch Ascension",
        "Doom Punch",
        "Royal Bow",
        "Blade Impact",
        "Eternal Slam",
        "Chaos Insanity",
        "Meteor Punch",
        "Wraith Swing",
        "Cyclone Arms",
        "Barrel Storm",
        "Panic State",
        "Madness",
        "Arm Seeker",
        "Blade Slash",
        "Chaos Arms",
        "Victory Dab",
        "Vortex Spin",
        "Phantom Dance",
        "Spin of Kings",
        "Lunar Dance",
        "Whirl of Fate",
        "Thriller Night",
        "Mecha Mode",
        "Silent Shuffle",
        "Smooth Groove",
        "Club Fever",
        "Sky Bounce",
        "Wild Frenzy",
        "Final Fall",
        "Undead Stride",
    }

    local function playAnimation()
        local char = AnimPlayer.Character or AnimPlayer.CharacterAdded:Wait()
        local hum  = char:FindFirstChildOfClass("Humanoid")
        if not hum then return end

        local animator = hum:FindFirstChildOfClass("Animator")
        if not animator then
            animator = Instance.new("Animator")
            animator.Parent = hum
        end

        if currentTrack then
            pcall(function() currentTrack:Stop() end)
            currentTrack = nil
        end

        local anim = Instance.new("Animation")
        anim.AnimationId = Animations[selectedAnimName]

        local ok, track = pcall(function()
            return animator:LoadAnimation(anim)
        end)
        anim:Destroy()
        if not ok or not track then return end

        currentTrack = track
        currentTrack.Priority = Enum.AnimationPriority.Action
        currentTrack.Looped   = true
        currentTrack:Play()
        currentTrack:AdjustSpeed(animSpeed)

        task.spawn(function()
            while animEnabled and currentTrack == track do
                if currentTrack and currentTrack.IsPlaying and currentTrack.TimePosition > 5 then
                    currentTrack.TimePosition = 0.3
                end
                task.wait(0.05)
            end
        end)
    end

    local function stopAnimation()
        if currentTrack then
            pcall(function() currentTrack:Stop() end)
            currentTrack = nil
        end
    end

    AnimationsGroup:AddToggle("EchoPlayAnimation", {
        Text = "Play Animation",
        Default = false,
        Callback = function(on)
            animEnabled = on and true or false
            if on then
                playAnimation()
            else
                stopAnimation()
            end
        end,
    })

    AnimationsGroup:AddDropdown("EchoAnimationSelect", {
        Text = "Animation",
        Values = AnimationList,
        Default = "Phantom Rush",
        Multi = false,
        Searchable = true,
        Callback = function(v)
            selectedAnimName = v
            if animEnabled then
                playAnimation()
            end
        end,
    })

    AnimationsGroup:AddSlider("EchoAnimationSpeed", {
        Text = "Speed",
        Default = 1,
        Min = 0.1,
        Max = 100,
        Rounding = 2,
        Suffix = "x",
        Callback = function(v)
            animSpeed = v
            if currentTrack and currentTrack.IsPlaying then
                pcall(function() currentTrack:AdjustSpeed(v) end)
            end
        end,
    })

    -- Re-apply on respawn if the toggle is on.
    AnimPlayer.CharacterAdded:Connect(function()
        task.wait(0.35)
        if animEnabled then
            playAnimation()
        end
    end)

    pcall(function()
        Library:OnUnload(function()
            animEnabled = false
            stopAnimation()
        end)
    end)
end

--grab tab
-- ============================================================
-- GRAB TAB (v4 — Cleaned Layout)
-- ============================================================
do
    local GW = Workspace
    local GRS = ReplicatedStorage
    local GDebris = Debris
    local GRun = RunService
    local GPlayers = Players
    local GUI = UserInputService

    local GrabEvents      = GRS:FindFirstChild("GrabEvents")
    local SetNetworkOwner = GrabEvents and GrabEvents:FindFirstChild("SetNetworkOwner")
    local DestroyGrabLine = GrabEvents and GrabEvents:FindFirstChild("DestroyGrabLine")
    local CreateGrabLine  = GrabEvents and GrabEvents:FindFirstChild("CreateGrabLine")
    local MenuToys        = GRS:FindFirstChild("MenuToys")
    local DestroyToy      = MenuToys and MenuToys:FindFirstChild("DestroyToy")
    local SpawnToyRemote  = MenuToys and MenuToys:FindFirstChild("SpawnToyRemoteFunction")

    -- ============================================================
    -- UI LAYOUT
    -- ============================================================
    local GrabBox      = Tabs.Grab:AddLeftTabbox("Grab")
    local GrabsTab     = GrabBox:AddTab("Grabs", "hand")
    local FunAurasTab  = GrabBox:AddTab("Fun Auras", "sparkles")

    local PowersBox    = Tabs.Grab:AddRightTabbox("Powers")
    local GrabPowers   = PowersBox:AddTab("Grab Powers", "sliders-horizontal")
    local FunPowers    = PowersBox:AddTab("Fun Aura Powers", "sliders-horizontal")

    -- ============================================================
    -- STATE
    -- ============================================================
    local S = {
        -- Grabs
        ThrowOnRelease   = false,
        KillGrab         = false,
        SpamGrab         = false,
        MasslessGrab     = false,
        HeavyGrab        = false,
        AnchorGrab       = false,
        -- Auras
        KillAura         = false,
        FlingAura        = false,
        HeavenAura       = false,
        VoidAura         = false,
        -- Powers
        ThrowPower       = 750,
        FlingPower       = 1,
        FlingTarget      = "Player",
        -- Aura ranges
        KillAuraRadius   = 25,
        FlingAuraRadius  = 30,   -- kept internal, slider removed
        HeavenAuraRadius = 25,
        VoidAuraRadius   = 25,
        -- Aura target modes
        KillAuraTarget   = "Player",
        HeavenAuraTarget = "Player",
        VoidAuraTarget   = "Player",
    }

    local Conns = {}
    local function Track(c) table.insert(Conns, c); return c end
    local function Disc(c) if c then pcall(function() c:Disconnect() end) end end

    -- ============================================================
    -- HELPERS
    -- ============================================================
    local function Root() local c = LocalPlayer.Character; return c and c:FindFirstChild("HumanoidRootPart") end
    local function Hum()  local c = LocalPlayer.Character; return c and c:FindFirstChildOfClass("Humanoid") end
    local function GrabParts() return GW:FindFirstChild("GrabParts") end
    local function GrabPart()  local g = GrabParts(); return g and g:FindFirstChild("GrabPart") end
    local function DragPart()  local g = GrabParts(); return g and g:FindFirstChild("DragPart") end
    local function GrabWeld()  local p = GrabPart(); return p and p:FindFirstChildOfClass("WeldConstraint") end
    local function GrabbedPart() local w = GrabWeld(); return w and w.Part1 end
    local function GrabbedModel() local p = GrabbedPart(); return p and p:FindFirstAncestorOfClass("Model") end
    local function GrabbedPlayer() local m = GrabbedModel(); return m and GPlayers:GetPlayerFromCharacter(m) end
    local function Dist(a, b) return (a - b).Magnitude end
    local function FireOwner(p) if p and SetNetworkOwner then pcall(function() SetNetworkOwner:FireServer(p, p.CFrame) end) end end
    local function FireLine(p)  if p and DestroyGrabLine then pcall(function() DestroyGrabLine:FireServer(p) end) end end

    local function IsLocal(part)
        if not part or not part:IsA("BasePart") then return false end
        local c = LocalPlayer.Character
        return c ~= nil and part:IsDescendantOf(c)
    end

    local function CollectNearby(origin, radius, includePlayers, includeObjects)
        local list = {}
        if includePlayers then
            for _, plr in ipairs(GPlayers:GetPlayers()) do
                if plr ~= LocalPlayer and plr.Character then
                    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
                    local h = plr.Character:FindFirstChildOfClass("Humanoid")
                    if hrp and h and h.Health > 0 and Dist(hrp.Position, origin) <= radius then
                        table.insert(list, { part = hrp, player = plr })
                    end
                end
            end
        end
        if includeObjects then
            local params = OverlapParams.new()
            params.FilterType = Enum.RaycastFilterType.Exclude
            params.FilterDescendantsInstances = { LocalPlayer.Character, GrabParts() }
            local parts = GW:GetPartBoundsInRadius(origin, radius, params)
            for _, p in ipairs(parts) do
                if p:IsA("BasePart") and not IsLocal(p) then
                    local m = p:FindFirstAncestorOfClass("Model")
                    local plr = m and GPlayers:GetPlayerFromCharacter(m)
                    local inMap = m and GW:FindFirstChild("Map") and m:IsDescendantOf(GW.Map)
                    if not plr and not inMap then
                        table.insert(list, { part = p, player = nil })
                    end
                end
            end
        end
        return list
    end

    -- ============================================================
    -- THROW ON RELEASE
    -- ============================================================
    local ThrowConn
    local function StopThrow() Disc(ThrowConn); ThrowConn = nil end
    local function StartThrow()
        StopThrow()
        ThrowConn = Track(GW.ChildAdded:Connect(function(obj)
            if obj.Name ~= "GrabParts" then return end
            task.spawn(function()
                local gp = obj:FindFirstChild("GrabPart")
                if not gp then return end
                local w = gp:FindFirstChildOfClass("WeldConstraint")
                if not w or not w.Part1 then return end
                local target = w.Part1
                local bv = Instance.new("BodyVelocity")
                bv.Name = "EchoThrowBV"
                bv.MaxForce = Vector3.zero
                bv.Parent = target
                local pc
                pc = obj:GetPropertyChangedSignal("Parent"):Connect(function()
                    if obj.Parent then return end
                    Disc(pc)
                    if not S.ThrowOnRelease then pcall(function() bv:Destroy() end); return end
                    local last = GUI:GetLastInputType()
                    if last == Enum.UserInputType.MouseButton2 then
                        local cam = GW.CurrentCamera
                        if cam then
                            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                            bv.Velocity = cam.CFrame.LookVector * S.ThrowPower
                            GDebris:AddItem(bv, 0.35)
                        else
                            bv:Destroy()
                        end
                    else
                        bv:Destroy()
                    end
                end)
            end)
        end))
    end

    -- ============================================================
    -- KILL GRAB
    -- ============================================================
    local KillGrabConn
    local function StartKillGrab()
        Disc(KillGrabConn)
        KillGrabConn = Track(GRun.Heartbeat:Connect(function()
            if not S.KillGrab then return end
            local model = GrabbedModel()
            if not model then return end
            local hum = model:FindFirstChildOfClass("Humanoid")
            if not hum or hum.Health <= 0 then return end
            local root = model:FindFirstChild("HumanoidRootPart") or model.PrimaryPart
            if not root then return end
            pcall(function()
                FireOwner(root)
                FireLine(root)
                root.CFrame = root.CFrame - Vector3.new(0, 500, 0)
                local bv = Instance.new("BodyVelocity")
                bv.Velocity = Vector3.new(0, -1e7, 0)
                bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                bv.Parent = root
                GDebris:AddItem(bv, 0.25)
                hum:ChangeState(Enum.HumanoidStateType.Dead)
            end)
        end))
    end

    -- ============================================================
    -- SPAM GRAB / LOOP GRAB
    -- ============================================================
    local SpamGrabThread
    local function FindNearestTarget()
        local r = Root()
        if not r then return nil end
        local best, bestD = nil, math.huge
        for _, plr in ipairs(GPlayers:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
                local h = plr.Character:FindFirstChildOfClass("Humanoid")
                if hrp and h and h.Health > 0 then
                    local d = Dist(hrp.Position, r.Position)
                    if d < bestD then best, bestD = hrp, d end
                end
            end
        end
        return best
    end

    local function StartSpamGrab()
        if SpamGrabThread then return end
        SpamGrabThread = task.spawn(function()
            while S.SpamGrab do
                local holding = GrabParts() ~= nil
                if not holding then
                    local target = FindNearestTarget()
                    if target then
                        pcall(function()
                            FireOwner(target)
                            task.wait(0.03)
                            if mouse1press then
                                mouse1press()
                                task.wait(0.05)
                                mouse1release()
                            end
                        end)
                    end
                else
                    local gp = GrabParts()
                    if gp and DestroyToy then
                        pcall(function() DestroyToy:FireServer(gp) end)
                    end
                end
                task.wait(0.1)
            end
            SpamGrabThread = nil
        end)
    end
    local function StopSpamGrab()
        if SpamGrabThread then pcall(task.cancel, SpamGrabThread); SpamGrabThread = nil end
    end

    -- ============================================================
    -- MASSLESS GRAB  (fixed — true massless + max align responsiveness)
    -- ============================================================
    local MasslessConn
    local MasslessBackup = {}

    local function ApplyMasslessToDragPart()
        local dp = DragPart()
        if not dp then return end
        local ap = dp:FindFirstChild("AlignPosition")
        local ao = dp:FindFirstChild("AlignOrientation")
        if ap then pcall(function()
            ap.MaxForce = math.huge
            ap.MaxVelocity = math.huge
            ap.Responsiveness = 200
        end) end
        if ao then pcall(function()
            ao.MaxTorque = math.huge
            ao.Responsiveness = 200
        end) end
    end

    local function ApplyMasslessToModel(model)
        if not model then return end
        for _, p in ipairs(model:GetDescendants()) do
            if p:IsA("BasePart") then
                if MasslessBackup[p] == nil then MasslessBackup[p] = p.Massless end
                p.Massless = true
            end
        end
    end

    local function RestoreMassless()
        for p, o in pairs(MasslessBackup) do
            if p and p.Parent then pcall(function() p.Massless = o end) end
        end
        table.clear(MasslessBackup)
    end

    local function StartMassless()
        Disc(MasslessConn)
        -- Immediate pass
        local m = GrabbedModel(); if m then ApplyMasslessToModel(m) end
        ApplyMasslessToDragPart()

        -- React to new grabs + keep align responsiveness at max
        MasslessConn = Track(GRun.Heartbeat:Connect(function()
            if not S.MasslessGrab then return end
            local m = GrabbedModel(); if m then ApplyMasslessToModel(m) end
            ApplyMasslessToDragPart()
        end))
    end

    -- ============================================================
    -- HEAVY OBJECT GRAB
    -- ============================================================
    local HeavyConn
    local function StartHeavy()
        Disc(HeavyConn)
        HeavyConn = Track(GRun.Heartbeat:Connect(function()
            if not S.HeavyGrab then return end
            local dp = DragPart()
            if not dp then return end
            local ap = dp:FindFirstChild("AlignPosition")
            local ao = dp:FindFirstChild("AlignOrientation")
            if ap then pcall(function()
                ap.MaxForce = math.huge; ap.MaxVelocity = math.huge
                if ap.Responsiveness < 200 then ap.Responsiveness = 200 end
            end) end
            if ao then pcall(function()
                ao.MaxTorque = math.huge
                if ao.Responsiveness < 200 then ao.Responsiveness = 200 end
            end) end
        end))
    end

    -- ============================================================
    -- ANCHOR GRAB  (fixed — reliable anchor/restore cycle)
    -- ============================================================
    local AnchorConn
    local AnchorBackup = {}

    local function RestoreAnchors()
        for p, o in pairs(AnchorBackup) do
            if p and p.Parent then pcall(function() p.Anchored = o end) end
        end
        table.clear(AnchorBackup)
    end

    local function StartAnchor()
        Disc(AnchorConn)
        AnchorConn = Track(GRun.Heartbeat:Connect(function()
            if not S.AnchorGrab then
                -- If the toggle was flipped off but we still have backups, restore once.
                if next(AnchorBackup) then RestoreAnchors() end
                return
            end

            local model = GrabbedModel()
            if not model then
                -- Released → restore everything we anchored
                if next(AnchorBackup) then RestoreAnchors() end
                return
            end

            for _, p in ipairs(model:GetDescendants()) do
                if p:IsA("BasePart") then
                    if AnchorBackup[p] == nil then
                        AnchorBackup[p] = p.Anchored
                    end
                    if not p.Anchored then p.Anchored = true end
                end
            end
        end))
    end

    -- ============================================================
    -- AUTO REPAIR TOYS (was Auto Repair All Toys button, now a keybound action)
    -- ============================================================
    local function AutoRepairToys()
        local folder = GW:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        if not folder then
            Library:Notify({ Title = "Echo", Description = "No toy folder found.", Time = 3 })
            return
        end

        local repaired = 0
        for _, toy in ipairs(folder:GetChildren()) do
            if toy:IsA("Model") then
                local primary = toy.PrimaryPart
                    or toy:FindFirstChild("SoundPart")
                    or toy:FindFirstChild("HoldPart")
                    or toy:FindFirstChildWhichIsA("BasePart", true)
                if primary then
                    local needsRepair = false
                    for _, p in ipairs(toy:GetDescendants()) do
                        if p:IsA("BasePart") and not p.Anchored then
                            local welded = false
                            for _, c in ipairs(p:GetChildren()) do
                                if c:IsA("WeldConstraint") or c:IsA("Weld") then
                                    welded = true
                                    break
                                end
                            end
                            if not welded then needsRepair = true break end
                        end
                    end

                    if needsRepair then
                        pcall(function()
                            for _, p in ipairs(toy:GetDescendants()) do
                                if p:IsA("BasePart") and p ~= primary then
                                    if not p:FindFirstChildOfClass("WeldConstraint") then
                                        local w = Instance.new("WeldConstraint")
                                        w.Part0 = primary
                                        w.Part1 = p
                                        w.Parent = p
                                    end
                                end
                            end
                        end)
                        repaired = repaired + 1
                    end
                end
            end
        end

        for _, toy in ipairs(folder:GetDescendants()) do
            if toy:IsA("BasePart") then
                pcall(function() FireOwner(toy) end)
            end
        end

        Library:Notify({
            Title = "Echo",
            Description = "Repaired " .. tostring(repaired) .. " toy(s).",
            Time = 3,
        })
    end

    -- ============================================================
    -- UNANCHOR ALL TOYS  (only works if something is anchored)
    -- ============================================================
    local function UnanchorAllToys()
        local folder = GW:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        if not folder then
            Library:Notify({ Title = "Echo", Description = "No toy folder found.", Time = 3 })
            return
        end

        -- Check first — only act if at least one part is anchored
        local anyAnchored = false
        for _, toy in ipairs(folder:GetDescendants()) do
            if toy:IsA("BasePart") and toy.Anchored then
                anyAnchored = true
                break
            end
        end

        if not anyAnchored then
            Library:Notify({
                Title = "Echo",
                Description = "No anchored toys found.",
                Time = 3,
            })
            return
        end

        local count = 0
        for _, toy in ipairs(folder:GetDescendants()) do
            if toy:IsA("BasePart") and toy.Anchored then
                pcall(function() toy.Anchored = false end)
                count = count + 1
            end
        end

        Library:Notify({
            Title = "Echo",
            Description = "Unanchored " .. tostring(count) .. " part(s).",
            Time = 3,
        })
    end

    -- ============================================================
    -- FUN AURAS
    -- ============================================================
    local KillAuraConn, HeavenConn, VoidConn
    local FlingAuraThread
    local HeavenCd, VoidCd = {}, {}

    local function StartKillAura()
        Disc(KillAuraConn)
        KillAuraConn = Track(GRun.Heartbeat:Connect(function()
            if not S.KillAura then return end
            local r = Root(); if not r then return end
            local includeP = S.KillAuraTarget == "Player" or S.KillAuraTarget == "Both"
            local includeO = S.KillAuraTarget == "Object" or S.KillAuraTarget == "Both"
            for _, entry in ipairs(CollectNearby(r.Position, S.KillAuraRadius, includeP, includeO)) do
                local target = entry.part
                pcall(function()
                    FireOwner(target)
                    FireLine(target)
                end)
                task.wait()
                local hum = entry.player and entry.player.Character
                    and entry.player.Character:FindFirstChildOfClass("Humanoid")
                    or (target.Parent and target.Parent:FindFirstChildOfClass("Humanoid"))
                if hum and hum.Health > 0 then
                    pcall(function()
                        local bv = Instance.new("BodyVelocity")
                        bv.Velocity = Vector3.new(0, -1e7, 0)
                        bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                        bv.Parent = target
                        GDebris:AddItem(bv, 0.2)
                        hum:ChangeState(Enum.HumanoidStateType.Dead)
                    end)
                end
            end
        end))
    end

    local function StartFlingAura()
        if FlingAuraThread then return end
        FlingAuraThread = task.spawn(function()
            while S.FlingAura do
                local r = Root()
                if r then
                    local origin = r.Position
                    local includeP = S.FlingTarget == "Player" or S.FlingTarget == "Both"
                    local includeO = S.FlingTarget == "Object" or S.FlingTarget == "Both"
                    for _, entry in ipairs(CollectNearby(origin, S.FlingAuraRadius, includeP, includeO)) do
                        local p = entry.part
                        pcall(function()
                            FireOwner(p)
                            local dir = p.Position - origin
                            if dir.Magnitude < 0.01 then dir = Vector3.new(0, 1, 0) end
                            local bv = Instance.new("BodyVelocity")
                            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                            bv.Velocity = (dir.Unit + Vector3.new(0, 0.15, 0)) * (S.FlingPower * 500)
                            bv.P = 12500
                            bv.Parent = p
                            GDebris:AddItem(bv, 0.15)
                        end)
                    end
                end
                task.wait(0.05)
            end
            FlingAuraThread = nil
        end)
    end

    local function StartHeaven()
        Disc(HeavenConn)
        HeavenConn = Track(GRun.Heartbeat:Connect(function()
            if not S.HeavenAura then return end
            local r = Root(); if not r then return end
            local includeP = S.HeavenAuraTarget == "Player" or S.HeavenAuraTarget == "Both"
            local includeO = S.HeavenAuraTarget == "Object" or S.HeavenAuraTarget == "Both"
            for _, entry in ipairs(CollectNearby(r.Position, S.HeavenAuraRadius, includeP, includeO)) do
                local p = entry.part
                local key = entry.player or p
                local last = HeavenCd[key] or 0
                if os.clock() - last >= 1 then
                    HeavenCd[key] = os.clock()
                    pcall(function()
                        FireOwner(p)
                        FireLine(p)
                        p.CFrame = CFrame.new(0, 1000000, 0)
                        local bv = Instance.new("BodyVelocity")
                        bv.MaxForce = Vector3.new(0, math.huge, 0)
                        bv.Velocity = Vector3.new(0, 1000000, 0)
                        bv.Parent = p
                        GDebris:AddItem(bv, 0.5)
                    end)
                end
            end
        end))
    end

    local function StartVoid()
        Disc(VoidConn)
        VoidConn = Track(GRun.Heartbeat:Connect(function()
            if not S.VoidAura then return end
            local r = Root(); if not r then return end
            local includeP = S.VoidAuraTarget == "Player" or S.VoidAuraTarget == "Both"
            local includeO = S.VoidAuraTarget == "Object" or S.VoidAuraTarget == "Both"
            for _, entry in ipairs(CollectNearby(r.Position, S.VoidAuraRadius, includeP, includeO)) do
                local p = entry.part
                local key = entry.player or p
                local last = VoidCd[key] or 0
                if os.clock() - last >= 1 then
                    VoidCd[key] = os.clock()
                    pcall(function()
                        FireOwner(p)
                        FireLine(p)
                        p.CFrame = CFrame.new(0, -1000000, 0)
                        local bv = Instance.new("BodyVelocity")
                        bv.MaxForce = Vector3.new(0, math.huge, 0)
                        bv.Velocity = Vector3.new(0, -1000000, 0)
                        bv.Parent = p
                        GDebris:AddItem(bv, 0.5)
                    end)
                end
            end
        end))
    end

    -- ============================================================
    -- UI — GRABS TAB
    -- ============================================================
    GrabsTab:AddToggle("EchoThrowOnRelease", {
        Text = "Throw on Release",
        Default = false,
        Callback = function(v)
            S.ThrowOnRelease = v
            if v then StartThrow() else StopThrow() end
        end,
    })

    GrabsTab:AddToggle("EchoKillGrab", {
        Text = "Kill Grab",
        Default = false,
        Tooltip = "Instantly kills whatever you're currently holding",
        Callback = function(v)
            S.KillGrab = v
            if v then StartKillGrab() else Disc(KillGrabConn); KillGrabConn = nil end
        end,
    })

    GrabsTab:AddToggle("EchoSpamGrab", {
        Text = "Spam Grab / Loop Grab",
        Default = false,
        Tooltip = "Continuously grabs and releases the nearest player",
        Callback = function(v)
            S.SpamGrab = v
            if v then StartSpamGrab() else StopSpamGrab() end
        end,
    })

    GrabsTab:AddToggle("EchoMasslessGrab", {
        Text = "Massless Grab",
        Default = false,
        Tooltip = "Zeroes mass AND forces drag responsiveness to max — works on heavy/props",
        Callback = function(v)
            S.MasslessGrab = v
            if v then StartMassless() else Disc(MasslessConn); MasslessConn = nil; RestoreMassless() end
        end,
    })

    GrabsTab:AddToggle("EchoHeavyGrab", {
        Text = "Heavy Object Grab",
        Default = false,
        Callback = function(v)
            S.HeavyGrab = v
            if v then StartHeavy() else Disc(HeavyConn); HeavyConn = nil end
        end,
    })

    GrabsTab:AddToggle("EchoAnchorGrab", {
        Text = "Anchor Grab",
        Default = false,
        Tooltip = "Freezes the grabbed object in place while held",
        Callback = function(v)
            S.AnchorGrab = v
            if v then StartAnchor() else Disc(AnchorConn); AnchorConn = nil; RestoreAnchors() end
        end,
    })

    -- Auto Repair Toys — press-style keybind on a Label so Obsidian accepts it
    local AutoRepairLabel = GrabsTab:AddLabel("Auto Repair Toys Keybind")
    AutoRepairLabel:AddKeyPicker("EchoAutoRepairToysKey", {
        Text = "Auto Repair Toys Keybind",
        Default = "R",
        Mode = "Press",
        NoUI = false,
        SyncToggleState = false,
        Callback = function()
            AutoRepairToys()
        end,
    })

    -- Unanchor All Toys — plain button, only acts when something is anchored
    GrabsTab:AddButton({
        Text = "Unanchor All Toys",
        Tooltip = "Unanchors every anchored part inside your toy folder (no-op if none are anchored)",
        Func = function()
            UnanchorAllToys()
        end,
    })

    -- ============================================================
    -- UI — FUN AURAS TAB
    -- ============================================================
    FunAurasTab:AddToggle("EchoKillAura", {
        Text = "Kill Aura",
        Default = false,
        Callback = function(v)
            S.KillAura = v
            if v then StartKillAura() else Disc(KillAuraConn); KillAuraConn = nil end
        end,
    })

    FunAurasTab:AddToggle("EchoFlingAura", {
        Text = "Fling Aura",
        Default = false,
        Callback = function(v)
            S.FlingAura = v
            if v then StartFlingAura() end
        end,
    })

    FunAurasTab:AddToggle("EchoHeavenAura", {
        Text = "Heaven Aura",
        Default = false,
        Callback = function(v)
            S.HeavenAura = v
            if v then StartHeaven() else Disc(HeavenConn); HeavenConn = nil; table.clear(HeavenCd) end
        end,
    })

    FunAurasTab:AddToggle("EchoVoidAura", {
        Text = "Void Aura",
        Default = false,
        Callback = function(v)
            S.VoidAura = v
            if v then StartVoid() else Disc(VoidConn); VoidConn = nil; table.clear(VoidCd) end
        end,
    })

    -- ============================================================
    -- UI — GRAB POWERS TAB
    -- ============================================================
    GrabPowers:AddSlider("EchoThrowPower", {
        Text = "Throw Power",
        Default = 750, Min = 1, Max = 20000, Rounding = 0,
        Callback = function(v) S.ThrowPower = v end,
    })

    GrabPowers:AddSlider("EchoFlingPower", {
        Text = "Fling Aura Power",
        Default = 1, Min = 1, Max = 20, Rounding = 0,
        Callback = function(v) S.FlingPower = v end,
    })

    GrabPowers:AddDropdown("EchoFlingTarget", {
        Text = "Fling Target",
        Values = { "Player", "Object", "Both" },
        Default = "Player",
        Callback = function(v) S.FlingTarget = v end,
    })

    -- ============================================================
    -- UI — FUN AURA POWERS TAB (section titles removed, fling range removed)
    -- ============================================================
    FunPowers:AddSlider("EchoKillAuraRadius", {
        Text = "Kill Aura Range",
        Default = 25, Min = 5, Max = 100, Rounding = 0, Suffix = " st",
        Callback = function(v) S.KillAuraRadius = v end,
    })
    FunPowers:AddDropdown("EchoKillAuraTarget", {
        Text = "Kill Aura Target",
        Values = { "Player", "Object", "Both" },
        Default = "Player",
        Callback = function(v) S.KillAuraTarget = v end,
    })

    FunPowers:AddDivider()

    FunPowers:AddSlider("EchoHeavenAuraRadius", {
        Text = "Heaven Aura Range",
        Default = 25, Min = 5, Max = 100, Rounding = 0, Suffix = " st",
        Callback = function(v) S.HeavenAuraRadius = v end,
    })
    FunPowers:AddDropdown("EchoHeavenAuraTarget", {
        Text = "Heaven Aura Target",
        Values = { "Player", "Object", "Both" },
        Default = "Player",
        Callback = function(v) S.HeavenAuraTarget = v end,
    })

    FunPowers:AddDivider()

    FunPowers:AddSlider("EchoVoidAuraRadius", {
        Text = "Void Aura Range",
        Default = 25, Min = 5, Max = 100, Rounding = 0, Suffix = " st",
        Callback = function(v) S.VoidAuraRadius = v end,
    })
    FunPowers:AddDropdown("EchoVoidAuraTarget", {
        Text = "Void Aura Target",
        Values = { "Player", "Object", "Both" },
        Default = "Player",
        Callback = function(v) S.VoidAuraTarget = v end,
    })

    -- ============================================================
    -- CLEANUP
    -- ============================================================
    pcall(function()
        Library:OnUnload(function()
            S.ThrowOnRelease  = false
            S.KillGrab        = false
            S.SpamGrab        = false
            S.MasslessGrab    = false
            S.HeavyGrab       = false
            S.AnchorGrab      = false
            S.KillAura        = false
            S.FlingAura       = false
            S.HeavenAura      = false
            S.VoidAura        = false

            StopThrow()
            Disc(KillGrabConn)
            StopSpamGrab()
            Disc(MasslessConn)
            Disc(HeavyConn)
            Disc(AnchorConn)
            Disc(KillAuraConn)
            Disc(HeavenConn)
            Disc(VoidConn)
            RestoreMassless()
            RestoreAnchors()

            for _, c in ipairs(Conns) do
                pcall(function() c:Disconnect() end)
            end
            table.clear(Conns)
        end)
    end)
end

--endgrab tab

--defence tab
-- ============================================================
-- DEFENCE TAB
-- ============================================================
do
    local DefenceTab = Tabs.Defence
    local P = LocalPlayer
    local RS = ReplicatedStorage
    local WS = Workspace
    local RunService = game:GetService("RunService")
    local Players = game:GetService("Players")
    local Debris = game:GetService("Debris")
    local Stats = game:GetService("Stats")

    local GE = RS:WaitForChild("GrabEvents")
    local CE = RS:WaitForChild("CharacterEvents")
    local MT = RS:WaitForChild("MenuToys")
    local PE = RS:WaitForChild("PlayerEvents")

    local SetNetOwner = GE:WaitForChild("SetNetworkOwner")
    local StruggleEvent = CE:WaitForChild("Struggle")
    local RagdollRemote = CE:WaitForChild("RagdollRemote")
    local SpawnToyRemote = MT:WaitForChild("SpawnToyRemoteFunction")
    local DestroyToy = MT:WaitForChild("DestroyToy")
    local StickyEvent = PE:WaitForChild("StickyPartEvent")

    function dhrp()
        local c = P.Character
        return c and c:FindFirstChild("HumanoidRootPart")
    end

    function dhum()
        local c = P.Character
        return c and c:FindFirstChildOfClass("Humanoid")
    end

    function stvel(hrp)
        if hrp then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end
    end

    function getDistance(pos1, pos2)
        return (pos1 - pos2).Magnitude
    end

    function fireNetworkOwner(part)
        if part then
            pcall(function() SetNetOwner:FireServer(part, part.CFrame) end)
        end
    end

    function fireDestroyLine(part)
        if part then
            pcall(function() DestroyToy:FireServer(part) end)
        end
    end

    function spawnToy(name, cf, vec)
        local f = WS:FindFirstChild(P.Name .. "SpawnedInToys")
        if not f then return nil end

        local found
        local con
        con = f.ChildAdded:Connect(function(x)
            if x.Name == name then
                found = x
                if con then con:Disconnect() end
            end
        end)

        pcall(function() SpawnToyRemote:InvokeServer(name, cf, vec or Vector3.zero) end)

        local t = tick()
        while not found and tick() - t < 4 do
            RunService.Heartbeat:Wait()
        end

        if con then con:Disconnect() end
        return found
    end

    function toyFolder()
        return WS:FindFirstChild(P.Name .. "SpawnedInToys")
    end

    function checkNetworkOwner(part)
        local po = part:FindFirstChild("PartOwner")
        return po and po.Value == P.Name
    end

    local whitelistFriendsEnabled = false
    local MainProtection = InvincibilityGroup

    _G._EchoDefence = {
        P = P,
        RS = RS,
        WS = WS,
        SetNetOwner = SetNetOwner,
        StruggleEvent = StruggleEvent,
        RagdollRemote = RagdollRemote,
        SpawnToyRemote = SpawnToyRemote,
        DestroyToy = DestroyToy,
        StickyEvent = StickyEvent,
        MainProtection = MainProtection,
        dhrp = dhrp,
        dhum = dhum,
        stvel = stvel,
        getDistance = getDistance,
        fireNetworkOwner = fireNetworkOwner,
        fireDestroyLine = fireDestroyLine,
        spawnToy = spawnToy,
        toyFolder = toyFolder,
        checkNetworkOwner = checkNetworkOwner,
    }
end

-- BLOCK 2: ANTI GRAB
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local RS = E.RS
    local StruggleEvent = E.StruggleEvent
    local RagdollRemote = E.RagdollRemote
    local DestroyToy = E.DestroyToy
    local MainProtection = E.MainProtection
    local Players = game:GetService("Players")
    local ContextActionService = game:GetService("ContextActionService")
    local AntiGrabRunService = game:GetService("RunService")

    local antiGrabEnabled = false
    local antiGrabMode = "No Ragdoll"

    function isHeld()
        local v = P:FindFirstChild("IsHeld")
        return v ~= nil and v.Value == true
    end

    -- ============================================================
    -- ANTI GRAB — SHARED HELPERS
    -- ============================================================
    function ownerIsSet(owner)
        if not owner or not owner:IsA("StringValue") then return false end
        local ok, value = pcall(function() return owner.Value end)
        return ok and type(value) == "string" and value ~= ""
    end

    -- true when ANY of our parts currently carries a non-empty PartOwner
    function headOwned(head)
        local char = head and head.Parent
        if not char then return false end
        for _, object in ipairs(char:GetDescendants()) do
            if object.Name == "PartOwner" and ownerIsSet(object) then
                return true
            end
        end
        return false
    end

    function characterIsGrabbed(char)
        if not char or char ~= P.Character then return false end
        return isHeld() or headOwned(char:FindFirstChild("Head"))
    end

    function antiGrabAlive(state, token, char)
        return state.active
            and state._token == token
            and P.Character == char
            and char.Parent ~= nil
    end

    function disconnectAntiGrabConnections(state)
        for _, connection in pairs(state.conns or {}) do
            pcall(function() connection:Disconnect() end)
        end
        state.conns = {}
    end

    function disableRagdoll(char)
        for _, v in pairs(char:GetChildren()) do
            if v:IsA("BasePart") and v:FindFirstChild("BallSocketConstraint") and v.Name ~= "Head" then
                pcall(function() v.BallSocketConstraint.Enabled = false end)
                if v:FindFirstChild("RagdollLimbPart") then
                    pcall(function() v.RagdollLimbPart.WeldConstraint.Enabled = false end)
                end
            end
        end
    end

    function enableRagdoll(char)
        for _, v in pairs(char:GetChildren()) do
            if v:IsA("BasePart") and v:FindFirstChild("BallSocketConstraint") and v.Name ~= "Head" then
                pcall(function() v.BallSocketConstraint.Enabled = true end)
                if v:FindFirstChild("RagdollLimbPart") then
                    pcall(function() v.RagdollLimbPart.WeldConstraint.Enabled = true end)
                end
            end
        end
    end

    -- restores HRP anchor / humanoid flags AND ragdoll joints (no lingering disabled limbs)
    function restoreAntiGrabCharacter(char)
        if not char then return end
        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hrp then pcall(function() hrp.Anchored = false end) end
        if hum then
            pcall(function()
                hum.PlatformStand = false
                hum.AutoRotate = true
            end)
        end
        pcall(enableRagdoll, char)
    end

    function weldOn(w)
        if not w then return false end
        local ok, res = pcall(function()
            if w.Parent == nil then return false end
            return w.Enabled
        end)
        return ok and res or false
    end

    function destroyGrabLineRemote()
        local ge = RS:FindFirstChild("GrabEvents")
        return ge and ge:FindFirstChild("DestroyGrabLine")
    end

    function clearGrabLines()
        local remote = destroyGrabLineRemote()
        if not remote then return end
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= P and player.Character then
                local character = player.Character
                local root = character:FindFirstChild("HumanoidRootPart")
                local head = character:FindFirstChild("Head")
                if root then pcall(function() remote:FireServer(root) end) end
                if head then pcall(function() remote:FireServer(head) end) end
                for _, object in ipairs(character:GetDescendants()) do
                    if object.Name == "PartOwner" then
                        pcall(function() remote:FireServer(object.Parent) end)
                    end
                end
            end
        end
    end

    function seatKind()
        local hum = P.Character and P.Character:FindFirstChildOfClass("Humanoid")
        if not (hum and hum.SeatPart) then return nil end
        local o = hum.SeatPart
        local seen = {}
        while o do
            if seen[o] then break end
            seen[o] = true
            if o.Name == "CreatureBlobman" then return "blob" end
            if o.Parent and o.Parent.Name == (P.Name .. "SpawnedInToys") then return "own" end
            o = o.Parent
        end
        return nil
    end

    function onBlobSeat()
        return seatKind() == "blob"
    end

    function onProtectedSeat()
        return seatKind() ~= nil
    end

    -- ============================================================
    -- ANTI GRAB MODE: NO RAGDOLL  (xocu)
    -- ============================================================
    local xocu = { conns = {}, proc = false, active = false }

    function xocuClean()
        xocu.active = false
        xocu.proc = false
        xocu._token = (xocu._token or 0) + 1
        disconnectAntiGrabConnections(xocu)
        restoreAntiGrabCharacter(P.Character)
    end

    function xocuApply(char)
        if not char then return end
        xocu._token = (xocu._token or 0) + 1
        local token = xocu._token
        disconnectAntiGrabConnections(xocu)
        xocu.active = true
        xocu.proc = false

        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        local head = char:FindFirstChild("Head")
        if not (hrp and hum and head) then
            task.spawn(function()
                char:WaitForChild("HumanoidRootPart", 5)
                char:WaitForChild("Humanoid", 5)
                char:WaitForChild("Head", 5)
                if antiGrabAlive(xocu, token, char) then xocuApply(char) end
            end)
            return
        end

        if onBlobSeat() then
            pcall(enableRagdoll, char)
        elseif not onProtectedSeat() then
            pcall(disableRagdoll, char)
        end

        local function beginEscape()
            if xocu.proc then return end
            if not antiGrabAlive(xocu, token, char) then return end
            if not characterIsGrabbed(char) then return end
            xocu.proc = true
            task.spawn(function()
                pcall(function()
                    hrp.Anchored = true
                    while antiGrabAlive(xocu, token, char) and characterIsGrabbed(char) do
                        pcall(function()
                            StruggleEvent:FireServer(P)
                            RagdollRemote:FireServer(hrp, 0)
                            hum.PlatformStand = false
                            hum.Sit = false
                            hum.AutoRotate = true
                            if hum.MoveDirection.Magnitude > 0 then
                                hrp.CFrame = hrp.CFrame + hum.MoveDirection * (hum.WalkSpeed / 60)
                            end
                            if weldOn(hrp:FindFirstChild("WeldHRP")) then
                                head.CFrame = hrp.CFrame + Vector3.new(0, 1.35, 0)
                            end
                        end)
                        task.wait()
                    end
                end)
                pcall(function() hrp.Anchored = false end)
                if xocu._token == token then xocu.proc = false end
                if P.Character == char and not characterIsGrabbed(char) then
                    task.spawn(clearGrabLines)
                end
            end)
        end

        xocu.conns.Monitor = AntiGrabRunService.Heartbeat:Connect(function()
            if not antiGrabAlive(xocu, token, char) then return end
            local ragdolled = hum:FindFirstChild("Ragdolled")
            if ragdolled and ragdolled.Value and not onBlobSeat() then
                pcall(disableRagdoll, char)
                pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
            end
            beginEscape()
        end)

        beginEscape()
    end

    function xocuStart()
        xocuClean()
        xocuApply(P.Character)
    end

    -- ============================================================
    -- ANTI GRAB MODE: RAGDOLL  (nrd)
    -- ============================================================
    local nrd = { conns = {}, proc = false, active = false, walk = false }

    function nrdDisableLimbs(char)
        for _, v in pairs(char:GetChildren()) do
            if v:IsA("BasePart") and v:FindFirstChild("BallSocketConstraint") and v.Name ~= "Head" then
                pcall(function() v.BallSocketConstraint.Enabled = false end)
                if v:FindFirstChild("RagdollLimbPart") then
                    pcall(function() v.RagdollLimbPart.WeldConstraint.Enabled = false end)
                end
            end
        end
    end

    function nrdClean()
        nrd.active = false
        nrd.proc = false
        nrd.walk = false
        nrd._token = (nrd._token or 0) + 1
        disconnectAntiGrabConnections(nrd)
        restoreAntiGrabCharacter(P.Character)
    end

    function nrdApply(char)
        if not char then return end
        nrd._token = (nrd._token or 0) + 1
        local token = nrd._token
        disconnectAntiGrabConnections(nrd)
        nrd.active = true
        nrd.proc = false
        nrd.walk = false

        local hrp = char:FindFirstChild("HumanoidRootPart")
        local hum = char:FindFirstChildOfClass("Humanoid")
        local head = char:FindFirstChild("Head")
        if not (hrp and hum and head) then
            task.spawn(function()
                char:WaitForChild("HumanoidRootPart", 5)
                char:WaitForChild("Humanoid", 5)
                char:WaitForChild("Head", 5)
                if antiGrabAlive(nrd, token, char) then nrdApply(char) end
            end)
            return
        end

        if not onProtectedSeat() then
            nrdDisableLimbs(char)
        end

        local function beginEscape()
            if nrd.proc then return end
            if not antiGrabAlive(nrd, token, char) then return end
            if not characterIsGrabbed(char) then return end
            nrd.proc = true
            task.spawn(function()
                pcall(function()
                    hrp.Anchored = false
                    while antiGrabAlive(nrd, token, char) and characterIsGrabbed(char) do
                        pcall(function()
                            StruggleEvent:FireServer(P)
                            RagdollRemote:FireServer(hrp, 0)
                            hum.Sit = false
                            hum.AutoRotate = true
                            if hum.MoveDirection.Magnitude > 0 then
                                hrp.CFrame = hrp.CFrame + hum.MoveDirection * 0.43
                            end
                            if weldOn(hrp:FindFirstChild("WeldHRP")) then
                                head.CFrame = hrp.CFrame + Vector3.new(0, 1.35, 0)
                            end
                            local ragdolled = hum:FindFirstChild("Ragdolled")
                            if ragdolled and ragdolled.Value and not onBlobSeat() then
                                nrdDisableLimbs(char)
                            end
                        end)
                        task.wait()
                    end
                end)
                pcall(function() hrp.Anchored = false end)
                if nrd._token == token then nrd.proc = false end
                if P.Character == char and not characterIsGrabbed(char) then
                    task.spawn(clearGrabLines)
                end
            end)
        end

        nrd.conns.Monitor = AntiGrabRunService.Heartbeat:Connect(function()
            if not antiGrabAlive(nrd, token, char) then return end
            local ragdolled = hum:FindFirstChild("Ragdolled")
            if ragdolled and ragdolled.Value and not onBlobSeat() then
                nrdDisableLimbs(char)
            end
            beginEscape()
        end)

        beginEscape()
    end

    function nrdStart()
        nrdClean()
        nrdApply(P.Character)
    end

    -- ============================================================
    -- ANTI GRAB MODE: ESCAPE EVERYTHING  (cheeky)
    -- ============================================================
    local ACGE = false
    local cheekyThread = nil

    function checkgrab(obj)
        if obj:FindFirstAncestor('Plots') or obj:FindFirstAncestor('Slots') or obj:FindFirstAncestor('Map') then
            return
        end
        local prt = obj:FindFirstChild('SoundPart') or obj:FindFirstChild('HumanoidRootPart')
            or obj:FindFirstChild('Main') or obj:FindFirstChild('Balloon')
        if obj:FindFirstChild('Humanoid') then
            if not obj.Humanoid.SeatPart then
                if prt and prt.ReceiveAge ~= 0 then
                    return false
                end
            elseif obj.Humanoid.SeatPart.Parent then
                return checkgrab(obj.Humanoid.SeatPart.Parent)
            end
        elseif prt and prt.ReceiveAge ~= 0 then
            return false
        end
        return true
    end

    function cheekyAntiGrab()
        local model
        while ACGE do
            local char = P.Character
            if char and char:FindFirstChild('Humanoid') then
                for _, track in ipairs(char.Humanoid:GetPlayingAnimationTracks()) do
                    if track.Animation.AnimationId == 'rbxassetid://7047322890' then
                        track:Stop()
                    end
                end
            end

            local toyFolder = WS:FindFirstChild(P.Name .. 'SpawnedInToys')
            if toyFolder then
                model = toyFolder:FindFirstChild('FoodMayonnaise')
            end

            if char and not checkgrab(char) then
                local hum = char:FindFirstChild('Humanoid')
                local ragdolled = hum and hum:FindFirstChild('Ragdolled') and hum.Ragdolled.Value

                if ragdolled or not model then
                    pcall(function() StruggleEvent:FireServer(P) end)
                    if hum then
                        for _, track in ipairs(hum:GetPlayingAnimationTracks()) do
                            if track.Animation.AnimationId == 'rbxassetid://7047322890' then
                                track:Stop()
                            end
                        end
                    end
                else
                    if model and model:FindFirstChild('HoldPart') then
                        local hold = model.HoldPart
                        if hold:FindFirstChild('HoldItemRemoteFunction') then
                            task.spawn(function()
                                pcall(function()
                                    hold.HoldItemRemoteFunction:InvokeServer(model, char)
                                end)
                            end)
                            pcall(function() DestroyToy:FireServer(model) end)
                            if hum then
                                hum:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
                                hum.AutoRotate = true
                                hum.Sit = false
                            end
                            pcall(function()
                                ContextActionService:UnbindAction('Escape')
                                ContextActionService:UnbindAction('JumpRemover')
                            end)
                        end
                    end
                end
            end

            task.wait()
        end
    end

    function cheekyCleanup()
        local toyFolder = WS:FindFirstChild(P.Name .. 'SpawnedInToys')
        if toyFolder then
            local model = toyFolder:FindFirstChild('FoodMayonnaise')
            if model then pcall(function() DestroyToy:FireServer(model) end) end
        end
        local char = P.Character
        if char then
            local hum = char:FindFirstChildOfClass('Humanoid')
            if hum then
                pcall(function() hum.Sit = false end)
                pcall(function() hum.PlatformStand = false end)
            end
        end
    end

    function cheekyStart()
        ACGE = true
        if cheekyThread then task.cancel(cheekyThread) end
        cheekyThread = task.spawn(cheekyAntiGrab)
    end

    function cheekyStop()
        ACGE = false
        if cheekyThread then
            task.cancel(cheekyThread)
            cheekyThread = nil
        end
        cheekyCleanup()
    end

    -- ============================================================
    -- MODE DISPATCH / UI
    -- ============================================================
    function stopAll()
        xocuClean()
        nrdClean()
        cheekyStop()
    end

    function startMode(m)
        if m == "No Ragdoll" then
            xocuStart()
        elseif m == "Ragdoll" then
            nrdStart()
        else
            cheekyStart()
        end
    end

    -- ══════════════════════════════════════════════════════════════
    -- UNIVERSAL SHIELD — constraint sweep + network-ownership reclaim
    -- Runs every Heartbeat when Anti Grab is enabled (both modes).
    -- Destroys any Weld / WeldConstraint / RopeConstraint /
    -- SpringConstraint that originates from another player's assembly.
    -- Also re-claims SetNetworkOwner on HRP every 60 frames.
    -- ══════════════════════════════════════════════════════════════

    local HOSTILE_CONSTRAINT_TYPES = {
        WeldConstraint = true,
        Weld = true,
        RopeConstraint = true,
        SpringConstraint = true,
        RodConstraint = true,
        BallSocketConstraint = true,
        HingeConstraint = true,
        AlignPosition = true,
        AlignOrientation = true,
        LinearVelocity = true,
        AngularVelocity = true,
        BodyVelocity = true,
        BodyAngularVelocity = true,
        BodyPosition = true,
        BodyGyro = true,
        BodyForce = true,
        VectorForce = true,
        Torque = true,
    }

    local universalShieldConn = nil
    local universalShieldFrame = 0

    local function isPartFromOtherPlayer(part)
        if not part or not part:IsA("BasePart") then return false end
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= P and p.Character and part:IsDescendantOf(p.Character) then
                return true
            end
        end
        return false
    end

    local function sweepConstraints(character)
        for _, desc in ipairs(character:GetDescendants()) do
            if HOSTILE_CONSTRAINT_TYPES[desc.ClassName] then
                local shouldNuke = false
                pcall(function()
                    if desc:IsA("WeldConstraint") or desc:IsA("Weld") then
                        shouldNuke = isPartFromOtherPlayer(desc.Part0)
                                  or isPartFromOtherPlayer(desc.Part1)
                    elseif desc.ClassName == "RopeConstraint"
                        or desc.ClassName == "SpringConstraint"
                        or desc.ClassName == "RodConstraint"
                        or desc.ClassName == "BallSocketConstraint"
                        or desc.ClassName == "HingeConstraint" then
                        local a0 = desc.Attachment0 and desc.Attachment0.Parent
                        local a1 = desc.Attachment1 and desc.Attachment1.Parent
                        shouldNuke = isPartFromOtherPlayer(a0)
                                  or isPartFromOtherPlayer(a1)
                    else
                        -- Movers: nuke in NoRagdoll when parent is external.
                        if antiGrabMode == "No Ragdoll" then
                            shouldNuke = not desc.Parent:IsDescendantOf(character)
                        else
                            shouldNuke = isPartFromOtherPlayer(desc.Parent)
                        end
                    end
                end)
                if shouldNuke then
                    pcall(function() desc:Destroy() end)
                end
            end
        end
    end

    local function startUniversalShield()
        if universalShieldConn then return end
        universalShieldConn = AntiGrabRunService.Heartbeat:Connect(function()
            if not antiGrabEnabled then return end
            local character = P.Character
            if not character then return end

            -- Constraint sweep every frame.
            sweepConstraints(character)

            -- NoRagdoll extras: keep state from flipping, clamp fling velocity.
            if antiGrabMode == "No Ragdoll" then
                local hum = character:FindFirstChildOfClass("Humanoid")
                local hrp = character:FindFirstChild("HumanoidRootPart")
                if hum then
                    if hum.PlatformStand then hum.PlatformStand = false end
                    if hum.Sit          then hum.Sit = false           end
                end
                if hrp and hrp.AssemblyLinearVelocity.Magnitude > 220 then
                    pcall(function()
                        hrp.AssemblyLinearVelocity = Vector3.zero
                    end)
                end
            end

            -- Network ownership reclaim every 60 frames.
            universalShieldFrame = universalShieldFrame + 1
            if universalShieldFrame >= 60 then
                universalShieldFrame = 0
                local hrp = character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    pcall(function() SetNetOwner:FireServer(hrp, hrp.CFrame) end)
                end
            end
        end)
    end

    local function stopUniversalShield()
        if universalShieldConn then
            pcall(function() universalShieldConn:Disconnect() end)
            universalShieldConn = nil
        end
        universalShieldFrame = 0
    end

    -- ── UI ────────────────────────────────────────────────────────

    MainProtection:AddToggle("AntiGrabMaster", {
        Text = "Anti Grab",
        Default = false,
        Callback = function(Value)
            antiGrabEnabled = Value
            stopAll()
            if Value then
                startMode(antiGrabMode)
                startUniversalShield()
            else
                stopUniversalShield()
            end
        end,
    })

    ConfigGroup:AddDropdown("AntiGrabMode", {
        Text = "Anti Grab Mode",
        Values = { "No Ragdoll", "Ragdoll", "Escape Everything" },
        Default = "No Ragdoll",
        Callback = function(m)
            antiGrabMode = m
            if antiGrabEnabled then
                stopAll()
                startMode(m)
            end
        end,
    })

    -- Re-hook on every respawn: re-apply mode AND restart universal shield.
    P.CharacterAdded:Connect(function()
        task.wait(0.5)
        if not antiGrabEnabled then return end
        -- Re-apply mode-specific protection.
        if antiGrabMode == "No Ragdoll" then
            xocuClean()
            xocuApply(P.Character)
        elseif antiGrabMode == "Ragdoll" then
            nrdClean()
            nrdApply(P.Character)
        else
            cheekyStart()
        end
        -- Restart the universal shield for the new character.
        stopUniversalShield()
        startUniversalShield()
    end)

    pcall(function()
        Library:OnUnload(function()
            antiGrabEnabled = false
            stopAll()
            stopUniversalShield()
        end)
    end)
end


-- BLOCK 3: ANTI GUCCI
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local SetNetOwner = E.SetNetOwner
    local StruggleEvent = E.StruggleEvent
    local RagdollRemote = E.RagdollRemote
    local SpawnToyRemote = E.SpawnToyRemote
    local DestroyToy = E.DestroyToy
    local StickyEvent = E.StickyEvent
    local MainProtection = E.MainProtection
    local dhrp = E.dhrp
    local dhum = E.dhum
    local RunService = game:GetService("RunService")

    local antiGucciMode = "Blobman"
    local antiGucciEnabled = false
    local antiGucciSession = 0
    local GUE = false
    local gucciRunning = false
    local itm = nil
    local sp, sv = nil, nil
    local hum, hrp = nil, nil
    local humconnection = nil
    local seatChangedConn = nil
    local inexistance = {}
    local charConn = nil
    local UP_1E15 = Vector3.new(0, 1e15, 0)
    local seatConn = nil
    local seatFor = nil
    local MI_CARPETA = P.Name .. "SpawnedInToys"
    local ALTO = CFrame.new(0, 100000000, 10)
    local CanSpawnToy = P:WaitForChild("CanSpawnToy")

    function getCharParts()
        local char = P.Character
        if not char then return nil, nil end
        return char:FindFirstChild("Humanoid"), char:FindFirstChild("HumanoidRootPart")
    end

    function cleanupGucci()
        GUE = false
        gucciRunning = false

        if charConn then
            charConn:Disconnect()
            charConn = nil
        end

        if seatConn then
            seatConn:Disconnect()
            seatConn = nil
        end
        seatFor = nil

        if gucciDestroyWatch then
            pcall(function() gucciDestroyWatch:Disconnect() end)
            gucciDestroyWatch = nil
        end
        gucciLastToy = nil
        gucciLastKnownPlayer = nil
        if itm and itm.Parent then
            pcall(function() if DestroyToy then DestroyToy:FireServer(itm) end end)
        end
        itm = nil
        inexistance = {}
        if seatChangedConn then
            seatChangedConn:Disconnect()
            seatChangedConn = nil
        end
        humconnection = nil
        sp = nil
        sv = nil

        local char = P.Character
        if char then
            local hum = char:FindFirstChild("Humanoid")
            if hum then
                hum.Sit = false
                hum.PlatformStand = false
            end
        end
    end

    function diddle()
        if not GUE then return end
        if itm then
            for _, prt in pairs(itm:GetChildren()) do
                if prt:IsA("BasePart") then
                    prt.CanCollide = false
                end
            end
        end
        if hrp then
            hrp.CFrame = sp
            hrp.AssemblyLinearVelocity = sv
        end
        if hum then
            seatChangedConn = hum:GetPropertyChangedSignal("SeatPart"):Once(diddle)
        end
    end

    function spawnBlobman(Session, SpawnCFrame)
        if not GUE or Session ~= antiGucciSession or not (CanSpawnToy and safeValue(CanSpawnToy, false)) then
            return
        end

        task.spawn(function()
            if GUE and Session == antiGucciSession and (CanSpawnToy and safeValue(CanSpawnToy, false)) then
                pcall(function()
                    SpawnToyRemote:InvokeServer("CreatureBlobman", SpawnCFrame, Vector3.zero)
                end)
            end
        end)
    end

    function onOccupant(seat)
        if not GUE or not hum or not hrp or hrp.Parent == nil then return end
        if seat.Occupant == hum or hum.SeatPart ~= nil then return end
        local rag = hum:FindFirstChild("Ragdolled")
        if rag and safeValue(rag, false) then return end
        seat:Sit(hum)
        RagdollRemote:FireServer(hrp, 0.016)
    end

    function gucciblob(Session)
        if gucciRunning then return end
        gucciRunning = true
        local TOY = "CreatureBlobman"
        local guccion = false
        local tickyticky = tick()
        local lastCleanup = tick()

        while GUE and Session == antiGucciSession do
            hum, hrp = getCharParts()
            if hum and hrp then
                local now = tick()
                local dt = now - tickyticky
                tickyticky = now

                local shouldSkip = false

                if itm and itm:FindFirstChild("VehicleSeat") then
                    local occupant = itm.VehicleSeat.Occupant
                    if occupant and occupant ~= hum then
                        task.spawn(function() pcall(function() DestroyToy:FireServer(itm) end) end)
                        itm = nil
                        guccion = false
                        inexistance = {}
                        task.wait(0.1)
                        spawnBlobman(Session, hrp.CFrame * ALTO)
                        task.wait()
                        shouldSkip = true
                    end
                end

                if not shouldSkip then
                    if itm and itm.Parent and not ((itm:FindFirstChild("HumanoidRootPart") or itm:FindFirstChild("SoundPart")) and itm:FindFirstChild("VehicleSeat") and (not itm.VehicleSeat.Occupant or itm.VehicleSeat.Occupant ~= hum) or inexistance[itm] < 1) then
                        task.spawn(function() pcall(function() DestroyToy:FireServer(itm) end) end)
                    else
                        local toyFolder = WS:FindFirstChild(MI_CARPETA)
                        itm = toyFolder and toyFolder:FindFirstChild(TOY)
                        if itm then
                            watchGucciToy(itm)
                        end
                    end

                    if not sp or not hum.SeatPart then
                        sp = hrp.CFrame
                        sv = hrp.AssemblyLinearVelocity
                    end

                    if humconnection ~= hum then
                        if seatChangedConn then
                            seatChangedConn:Disconnect()
                            seatChangedConn = nil
                        end
                        humconnection = hum
                        seatChangedConn = hum:GetPropertyChangedSignal("SeatPart"):Once(diddle)
                    end

                    local wait = true

                    if itm then
                        inexistance[itm] = (inexistance[itm] or 0) + dt
                        local raiz = itm:FindFirstChild("HumanoidRootPart") or itm:FindFirstChild("SoundPart")

                        if raiz and itm:FindFirstChild("VehicleSeat") and (not itm.VehicleSeat.Occupant or itm.VehicleSeat.Occupant == hum) then
                            if not guccion then
                                raiz.AssemblyLinearVelocity = Vector3.zero
                            end

                            local diddy = itm.VehicleSeat
                            if seatFor ~= diddy then
                                if seatConn then seatConn:Disconnect() end
                                seatFor = diddy
                                seatConn = diddy:GetPropertyChangedSignal("Occupant"):Connect(function()
                                    if GUE then onOccupant(diddy) end
                                end)
                            end
                            diddy.Parent = nil
                            diddy.Parent = itm

                            local miPos = hrp.CFrame.Position
                            if (miPos - raiz.CFrame.Position).Magnitude < 28 then
                                for _, prt in pairs(itm:GetChildren()) do
                                    if prt:IsA("BasePart") and prt.CanQuery then
                                        pcall(function() SetNetOwner:FireServer(prt, CFrame.lookAt(miPos, prt.CFrame.Position)) end)
                                    end
                                end
                            end

                            if not hum.SeatPart then
                                hum.Sit = false
                            else
                                wait = false
                                task.wait()
                            end

                            if hrp and hum and (itm:FindFirstChild("HumanoidRootPart") or itm:FindFirstChild("SoundPart")) and itm:FindFirstChild("VehicleSeat") and (not itm.VehicleSeat.Occupant or itm.VehicleSeat.Occupant == hum) then
                                local ragdolled = hum:FindFirstChild("Ragdolled")
                                if ragdolled and safeValue(ragdolled, false) then
                                    guccion = false
                                end

                                if itm.VehicleSeat.Occupant == hum then
                                    guccion = true
                                elseif not guccion then
                                    task.wait()
                                    wait = false
                                    if (itm:FindFirstChild("HumanoidRootPart") or itm:FindFirstChild("SoundPart")) and itm:FindFirstChild("VehicleSeat") and (not itm.VehicleSeat.Occupant or itm.VehicleSeat.Occupant == hum) then
                                        local ragdolled2 = hum:FindFirstChild("Ragdolled")
                                        if not ragdolled2 or not (safeValue(ragdolled2, false)) then
                                            itm.VehicleSeat:Sit(hum)
                                            RagdollRemote:FireServer(hrp, 0.016)
                                        end
                                    end
                                end

                                if guccion then
                                    (itm:FindFirstChild("HumanoidRootPart") or itm:FindFirstChild("SoundPart")).AssemblyLinearVelocity = UP_1E15
                                else
                                    if inexistance[itm] >= game.Stats.Network.ServerStatsItem["Data Ping"]:GetValue() / 250 then
                                        task.spawn(function() pcall(function() DestroyToy:FireServer(itm) end) end)
                                        hum.Sit = true
                                    end
                                end
                            end
                        else
                            if guccion then
                                hum.Sit = true
                            end
                            guccion = false
                            if inexistance[itm] >= 1 then
                                task.spawn(function() pcall(function() DestroyToy:FireServer(itm) end) end)
                                spawnBlobman(Session, hrp.CFrame * ALTO)
                            end
                        end
                    else
                        if guccion then
                            guccion = false
                            hum.Sit = true
                        end
                        if (CanSpawnToy and safeValue(CanSpawnToy, false)) then
                            spawnBlobman(Session, hrp.CFrame * ALTO)
                        end
                    end

                    if wait then
                        task.wait()
                        if hum then hum.Sit = false end
                    end
                end
            else
                task.wait()
            end

            if tick() - lastCleanup > 60 then
                lastCleanup = tick()
                for obj in pairs(inexistance) do
                    if not obj.Parent then
                        inexistance[obj] = nil
                    end
                end
            end
        end

        if Session == antiGucciSession then
            cleanupGucci()
            gucciRunning = false
        end
    end

    function onCharacterAddedBlob(char)
        local Session = antiGucciSession
        if not GUE then return end
        if itm and itm.Parent then
            pcall(function() DestroyToy:FireServer(itm) end)
        end
        itm = nil
        inexistance = {}
        if seatChangedConn then
            seatChangedConn:Disconnect()
            seatChangedConn = nil
        end
        humconnection = nil
        if seatConn then
            seatConn:Disconnect()
            seatConn = nil
        end
        seatFor = nil
        sp = nil
        sv = nil
        char:WaitForChild("Humanoid", 10)
        char:WaitForChild("HumanoidRootPart", 10)
        hum, hrp = getCharParts()
        task.wait(0.5)
        if GUE and Session == antiGucciSession and not gucciRunning then
            task.spawn(function()
                gucciblob(Session)
            end)
        end
    end

    local function watchGucciToy(toy)
        gucciLastToy = toy
        gucciLastKnownPlayer = nil
        if gucciDestroyWatch then
            pcall(function() gucciDestroyWatch:Disconnect() end)
            gucciDestroyWatch = nil
        end
        if not toy then return end

        gucciDestroyWatch = toy.AncestryChanged:Connect(function(_, parent)
            if parent ~= nil or not antiGucciEnabled or gucciLastToy ~= toy then
                return
            end

            local remover = gucciLastKnownPlayer
            pcall(function()
                if not remover and toy:FindFirstChild("VehicleSeat", true) then
                    local occupant = toy:FindFirstChild("VehicleSeat", true).Occupant
                    if occupant and occupant.Parent then
                        local player = Players:GetPlayerFromCharacter(occupant.Parent)
                        if player and player ~= P then remover = player end
                    end
                end
                if not remover then
                    for _, obj in ipairs(toy:GetDescendants()) do
                        if obj.Name == "PartOwner" and obj:IsA("StringValue") then
                            local player = Players:FindFirstChild(obj.Value)
                            if player and player ~= P then
                                remover = player
                                break
                            end
                        end
                    end
                end
            end)
            local desc
            if remover then
                desc = "You're Gucci Has been removed by " .. remover.DisplayName .. " (" .. remover.Name .. ")"
            else
                desc = "You're Gucci Has been removed"
            end
          _G.__EchoNotify(desc, "gucci_destroyed", 5, "GucciDestroyed")
            gucciLastToy = nil
        end)
    end

    local function stopGucciDestroyWatch()
        if gucciDestroyWatch then
            pcall(function() gucciDestroyWatch:Disconnect() end)
            gucciDestroyWatch = nil
        end
        gucciLastToy = nil
        gucciLastKnownPlayer = nil
    end

    function GucciBlobmanStart()
        if GUE then return end
        antiGucciSession = antiGucciSession + 1
        local Session = antiGucciSession
        GUE = true
        if charConn then charConn:Disconnect() end
        charConn = P.CharacterAdded:Connect(onCharacterAddedBlob)
        task.spawn(function()
            gucciblob(Session)
        end)
    end

    function GucciBlobmanStop()
        antiGucciSession = antiGucciSession + 1
        GUE = false
        cleanupGucci()
    end

    local gucciDestroyWatch = nil
    local gucciLastToy = nil
    local gucciLastKnownPlayer = nil

    local antiGucciConnectionTrain = nil
    local trainJumpConnection = nil
    local trainCharacterConnection = nil
    local safePositionTrain = nil
    local restoreFramesTrain = 0
    local autoGucciActiveTrain = false

    function startAntiGucciTrain()
        local character = P.Character or P.CharacterAdded:Wait()
        local humanoid = character:WaitForChild("Humanoid")
        local rootPart = character:WaitForChild("HumanoidRootPart")

        safePositionTrain = rootPart.Position

        local folder = WS:FindFirstChild("Map")
            and WS.Map:FindFirstChild("AlwaysHereTweenedObjects")
        local train = folder and folder:FindFirstChild("Train")
        local seat

        if train then
            for _, d in ipairs(train:GetDescendants()) do
                if d:IsA("Seat") then
                    seat = d
                    break
                end
            end
        end

        if seat then
            rootPart.CFrame = seat.CFrame + Vector3.new(0, 2, 0)
            seat:Sit(humanoid)
        end

        if trainJumpConnection then
            trainJumpConnection:Disconnect()
            trainJumpConnection = nil
        end

        trainJumpConnection = humanoid:GetPropertyChangedSignal("Jump"):Connect(function()
            if humanoid.Jump and humanoid.Sit then
                restoreFramesTrain = 15
                safePositionTrain = rootPart.Position
            end
        end)

        if antiGucciConnectionTrain then
            antiGucciConnectionTrain:Disconnect()
            antiGucciConnectionTrain = nil
        end

        antiGucciConnectionTrain = RunService.Heartbeat:Connect(function()
            if not rootPart.Parent or not humanoid.Parent then
                return
            end

            if RagdollRemote then
                pcall(function()
                    RagdollRemote:FireServer(rootPart, 0)
                end)
            end

            if restoreFramesTrain > 0 and safePositionTrain then
                rootPart.CFrame = CFrame.new(safePositionTrain)
                restoreFramesTrain = restoreFramesTrain - 1
            end
        end)

        task.spawn(function()
            while humanoid.Parent and humanoid.Sit do
                task.wait(1)
            end

            task.wait(0)

            if rootPart.Parent and safePositionTrain then
                rootPart.CFrame = CFrame.new(safePositionTrain)
            end
        end)
    end

    function stopAntiGucciTrain()
        if antiGucciConnectionTrain then
            antiGucciConnectionTrain:Disconnect()
            antiGucciConnectionTrain = nil
        end

        if trainJumpConnection then
            trainJumpConnection:Disconnect()
            trainJumpConnection = nil
        end

        restoreFramesTrain = 0
    end

    if trainCharacterConnection then
        trainCharacterConnection:Disconnect()
    end

    trainCharacterConnection = P.CharacterAdded:Connect(function()
        if autoGucciActiveTrain then
            task.wait(0.5)
            if autoGucciActiveTrain then
                startAntiGucciTrain()
            end
        end
    end)

    local tractorGucciActive = false
    local tractorGucciRunning = false
    local tractorItem = nil

    function tractorGucciLoop()
        if tractorGucciRunning then return end
        tractorGucciRunning = true
        while tractorGucciActive do
            pcall(function()
                local hum, hrp = dhum(), dhrp()
                if not hum or not hrp then return end
                local toyFolder = WS:FindFirstChild(P.Name .. "SpawnedInToys")
                tractorItem = toyFolder and toyFolder:FindFirstChild("TractorGreen")

                if not tractorItem and SpawnToyRemote then
                    local canSpawn = P:FindFirstChild("CanSpawnToy")
                    local ok, allowed = pcall(function() return canSpawn and canSpawn.Value end)
                    if ok and allowed then
                        pcall(function()
                            SpawnToyRemote:InvokeServer("TractorGreen", hrp.CFrame * CFrame.new(0, 100000000, 10), Vector3.zero)
                        end)
                        task.wait(0.5)
                        toyFolder = WS:FindFirstChild(P.Name .. "SpawnedInToys")
                        tractorItem = toyFolder and toyFolder:FindFirstChild("TractorGreen")
                    end
                end

                if tractorItem then
                    for _, prt in ipairs(tractorItem:GetDescendants()) do
                        if prt:IsA("BasePart") then
                            prt.CanCollide = false
                            prt.Transparency = 1
                            prt.CanTouch = false
                            prt.CanQuery = false
                        end
                    end
                    local seat = tractorItem:FindFirstChild("VehicleSeat", true)
                    if seat then
                        local occupant = seat.Occupant
                        if occupant and occupant ~= hum then
                            if DestroyToy then pcall(function() DestroyToy:FireServer(tractorItem) end) end
                            tractorItem = nil
                        elseif not occupant then
                            hrp.CFrame = seat.CFrame * CFrame.new(0, 1, 0)
                            seat:Sit(hum)
                        end
                    end
                end
            end)
            task.wait()
        end
        tractorGucciRunning = false
    end

    function stopTractorGucci()
        tractorGucciActive = false
        if tractorItem and tractorItem.Parent and DestroyToy then
            pcall(function() DestroyToy:FireServer(tractorItem) end)
        end
        tractorItem = nil
        tractorGucciRunning = false
    end

    P.CharacterAdded:Connect(function(char)
        if not tractorGucciActive then return end
        if tractorItem and tractorItem.Parent and DestroyToy then
            pcall(function() DestroyToy:FireServer(tractorItem) end)
        end
        tractorItem = nil
        tractorGucciRunning = false
        char:WaitForChild("Humanoid", 10)
        char:WaitForChild("HumanoidRootPart", 10)
        task.wait(0.5)
        task.spawn(tractorGucciLoop)
    end)

    local antiGucciToggle = MainProtection:AddToggle("AntiGucci", {
        Text = "Anti Grab (Gucci)",
        Default = false,
        Callback = function(v)
            antiGucciEnabled = v

            if not v then
                _G.__EchoNotify("Gucci has been Deactivated", "gucci_deactivated", 4, "GucciDestroyed")
                autoGucciActiveTrain = false
                GucciBlobmanStop()
                stopAntiGucciTrain()
                stopTractorGucci()

                local char = P.Character
                if char then
                    local hum = char:FindFirstChild("Humanoid")
                    if hum then
                        hum.Sit = false
                        hum.PlatformStand = false
                    end
                end
                return
            end

_G.__EchoNotify("Gucci has Been activated", "gucci_activated", 4, "GucciActivated")

            if antiGucciMode == "Blobman Gucci" then
                autoGucciActiveTrain = false
                stopTractorGucci()
                GucciBlobmanStart()
            elseif antiGucciMode == "Tractor Gucci (Invisible)" then
                autoGucciActiveTrain = false
                GucciBlobmanStop()
                stopAntiGucciTrain()
                tractorGucciActive = true
                task.spawn(tractorGucciLoop)
            elseif antiGucciMode == "Train Gucci (Invisible)" then
                GucciBlobmanStop()
                stopTractorGucci()
                autoGucciActiveTrain = true
                startAntiGucciTrain()
            end
        end
    })

    local seatlessGucciActive = false
    local seatlessGucciConnection = nil
    local seatlessGucciSpawnTick = nil

    MainProtection:AddToggle("SeatlessGucci", {
        Text = "Seatless Gucci",
        Default = false,
        Callback = function(Value)
            seatlessGucciActive = Value

            if Value then
                if not seatlessGucciConnection then
                    seatlessGucciConnection = RunService.RenderStepped:Connect(function()
                        if not seatlessGucciActive then return end

                        local char = P.Character
                        if not char then return end

                        local hum = char:FindFirstChildOfClass("Humanoid")
                        local root = char:FindFirstChild("HumanoidRootPart")
                        local myToys = WS:FindFirstChild(P.Name .. "SpawnedInToys")
                        local ocarina = myToys and myToys:FindFirstChild("InstrumentWoodwindOcarina")

                        if ocarina then
                            for _, prt in ipairs(char:GetChildren()) do
                                local owner = prt:FindFirstChild("PartOwner")
                                if owner and owner.Value ~= "" then
                                    local holdPart = ocarina:FindFirstChild("HoldPart")
                                    local holdFunc = holdPart and holdPart:FindFirstChild("HoldItemRemoteFunction")

                                    if holdFunc then
                                        task.spawn(function()
                                            pcall(function()
                                                holdFunc:InvokeServer(ocarina, char)
                                            end)
                                        end)

                                        if DestroyToy then
                                            pcall(function()
                                                DestroyToy:FireServer(ocarina)
                                            end)
                                        end

                                        if hum then
                                            hum:SetStateEnabled(Enum.HumanoidStateType.Jumping, true)
                                            hum.AutoRotate = true
                                            if hum.Sit then
                                                hum.Sit = false
                                            end
                                        end

                                        pcall(function()
                                            ContextActionService:UnbindAction("Escape")
                                            ContextActionService:UnbindAction("JumpRemover")
                                        end)

                                        owner.Value = ""
                                    end
                                end
                            end
                        else
                            local canSpawn = P:FindFirstChild("CanSpawnToy")
                            local canSpawnValue = false
                            pcall(function()
                                canSpawnValue = canSpawn and canSpawn.Value == true
                            end)

                            if canSpawnValue and not seatlessGucciSpawnTick then
                                seatlessGucciSpawnTick = tick()
                                task.spawn(function()
                                    if SpawnToyRemote then
                                        pcall(function()
                                            SpawnToyRemote:InvokeServer(
                                                "InstrumentWoodwindOcarina",
                                                CFrame.new(1e5, 1e5, 1e5),
                                                Vector3.zero
                                            )
                                        end)
                                    end
                                end)
                            elseif seatlessGucciSpawnTick
                                and tick() - seatlessGucciSpawnTick > 1
                                and not (myToys and myToys:FindFirstChild("InstrumentWoodwindOcarina")) then
                                seatlessGucciSpawnTick = nil
                            end

                            local grabbed = false
                            for _, prt in ipairs(char:GetChildren()) do
                                local owner = prt:FindFirstChild("PartOwner")
                                if owner and owner.Value ~= "" then
                                    grabbed = true
                                    break
                                end
                            end

                            if grabbed then
                                if StruggleEvent then
                                    pcall(function()
                                        StruggleEvent:FireServer(P)
                                    end)
                                end
                                if RagdollRemote and root then
                                    pcall(function()
                                        RagdollRemote:FireServer(root, 0)
                                    end)
                                end
                            end
                        end
                    end)
                end
            else
                if seatlessGucciConnection then
                    seatlessGucciConnection:Disconnect()
                    seatlessGucciConnection = nil
                end
                seatlessGucciSpawnTick = nil
            end
        end,
    })

    InvincibilityGroup:AddToggle("AntiInput", {
        Text = "Anti Input Lag",
        Default = false,
        Callback = function(v)
            if antiInputThread then
                task.cancel(antiInputThread)
                antiInputThread = nil
            end

            if not v then
                local inv = WS:FindFirstChild(P.Name .. "SpawnedInToys")
                if inv and inv:FindFirstChild("antiinputfood") then
                    pcall(function() if DestroyToy then DestroyToy:FireServer(inv.antiinputfood) end end)
                end
                return
            end

            antiInputThread = task.spawn(function()
                while Toggles.AntiInput and Toggles.AntiInput.Value do
                    pcall(function()
                        local char = P.Character
                        local hrp = char and char:FindFirstChild("HumanoidRootPart")
                        if hrp then
                            local toysFolder = WS:FindFirstChild(P.Name .. "SpawnedInToys")
                            local name = SelectedFood
                            local item = toysFolder and toysFolder:FindFirstChild(name)

                            for _, obj in pairs(WS:GetChildren()) do
                                if obj.Name == "Shuriken" and obj:IsA("Model") then
                                    for _, part in pairs(obj:GetDescendants()) do
                                        if part:IsA("BasePart") then
                                            part.CanCollide = false
                                            part.Massless = true
                                        end
                                    end
                                end
                            end

                            if not item or not item.Parent then
                                task.spawn(function()
                                    pcall(function()
                                        SpawnToyRemote:InvokeServer(name, hrp.CFrame * CFrame.new(0, -12, 0), Vector3.zero)
                                    end)
                                end)
                                task.wait(0.1)
                            else
                                local holdPart = item:FindFirstChild("HoldPart")
                                if holdPart then
                                    for _, v in pairs(item:GetDescendants()) do
                                        if v:IsA("BasePart") then
                                            v.CanCollide = false
                                            v.Massless = true
                                        end
                                    end
                                    task.spawn(function()
                                        pcall(function()
                                            holdPart.HoldItemRemoteFunction:InvokeServer(item, char)
                                        end)
                                    end)
                                    task.wait(getgenv().AntiInputHoldDuration or 0.02)
                                    task.spawn(function()
                                        pcall(function()
                                            holdPart.DropItemRemoteFunction:InvokeServer(
                                                item,
                                                CFrame.new(0, 5000, 0),
                                                Vector3.zero
                                            )
                                        end)
                                    end)
                                end
                            end
                        end
                    end)
                    task.wait(0.02)
                end
            end)
        end
    })

    ConfigGroup:AddDropdown("AntiGucciMode", {
        Text = "Anti Gucci Mode",
        Values = {"Blobman Gucci", "Tractor Gucci (Invisible)", "Train Gucci (Invisible)"},
        Default = "Blobman Gucci",
        Callback = function(mode)
            antiGucciMode = mode
            if antiGucciEnabled then
                antiGucciToggle:SetValue(false)
                task.wait(0.1)
                antiGucciToggle:SetValue(true)
            end
        end
    })

    local ToyList = {
        ["Coconut"] = "FoodCoconut",
        ["Banana"] = "FoodBanana",
        ["Fries"] = "FoodFrenchFries",
        ["MeatStick"] = "FoodMeatStick",
        ["Poop"] = "PoopPile",
        ["Donut"] = "FoodDonut",
        ["Cake"] = "FoodCakePink",
        ["Burger"] = "FoodHamburger",
        ["Pizza"] = "FoodPizzaCheese",
        ["Hotdog"] = "FoodHotdog",
        ["Mushroom"] = "FoodMushroomPoison",
        ["Banjo"] = "InstrumentGuitarBanjo",
        ["Violin"] = "InstrumentGuitarViolin",
        ["Ukulele"] = "InstrumentGuitarUkulele",
        ["Sax"] = "InstrumentWoodwindSaxophone",
        ["Vuvuzela"] = "InstrumentBrassVuvuzela",
        ["Bongos"] = "InstrumentDrumBongos",
        ["Mic"] = "InstrumentVoiceMicrophone",
        ["Pepperoni"] = "FoodPizzaPepperoni",
        ["Piano"] = "InstrumentPianoMelodica",
        ["Bread"] = "FoodBread",
        ["Egg"] = "FoodDippyEgg",
        ["Mayo"] = "FoodMayonnaise",
        ["WhiteMug"] = "CupMugWhite",
        ["Ocarina"] = "InstrumentWoodwindOcarina",
        ["SparklePoop"] = "PoopPileSparkle",
        ["BrownMug"] = "CupMugBrown",
        ["Trumpet"] = "InstrumentBrassTrumpet",
        ["Snare"] = "InstrumentDrumSnare",
        ["Lyre"] = "InstrumentGuitarLyre",
    }

    local DropdownValues = {}
    for shortName, _ in pairs(ToyList) do
        table.insert(DropdownValues, shortName)
    end
    table.sort(DropdownValues)

    SelectedFood = ToyList["Burger"]

    ConfigGroup:AddDropdown("EchoLoopTPMode", {
    Text = "Loop TP Mode",
    Values = {
        "None",
        "Loop TP (Random)",
        "Loop TP (OP / Tornado)",
    },
    Default = "Loop TP (Random)",
    Callback = function(Value)
        _G.__EchoLoopTPMode = Value
    end
})

local ToyImages = {
        ["Coconut"] = "rbxassetid://133875991045577",
        ["Banana"] = "rbxassetid://75484207768793",
        ["Fries"] = "rbxassetid://94745609579445",
        ["MeatStick"] = "rbxassetid://119912069009462",
        ["Poop"] = "rbxassetid://102977099237461",
        ["Donut"] = "rbxassetid://121685717166327",
        ["Cake"] = "rbxassetid://135967824239828",
        ["Burger"] = "rbxassetid://107005089698340",
        ["Pizza"] = "rbxassetid://106931956314027",
        ["Hotdog"] = "rbxassetid://106249136532377",
        ["Mushroom"] = "rbxassetid://103335679226894",
        ["Banjo"] = "rbxassetid://116000546776316",
        ["Violin"] = "rbxassetid://136796330128678",
        ["Ukulele"] = "rbxassetid://95555859846275",
        ["Sax"] = "rbxassetid://123311585597786",
        ["Vuvuzela"] = "rbxassetid://80041454248720",
        ["Bongos"] = "rbxassetid://121844247397293",
        ["Mic"] = "rbxassetid://122274381498776",
        ["Pepperoni"] = "rbxassetid://94216796644855",
        ["Piano"] = "rbxassetid://74776435919630",
        ["Bread"] = "rbxassetid://73483588123877",
        ["Egg"] = "rbxassetid://89197648899618",
        ["Mayo"] = "rbxassetid://87917161433070",
        ["WhiteMug"] = "rbxassetid://107681115846393",
        ["Ocarina"] = "rbxassetid://111465927393012",
        ["SparklePoop"] = "rbxassetid://121252324664942",
        ["BrownMug"] = "rbxassetid://74168393624840",
        ["Trumpet"] = "rbxassetid://96100498305663",
        ["Snare"] = "rbxassetid://106012654158373",
        ["Lyre"] = "rbxassetid://95132812667755",
    }

    ConfigGroup:AddDropdown("AntiInputToySelect", {
        Text = "Select Input Lag Toy",
        Values = DropdownValues,
        ValueImages = ToyImages,
        Default = "Burger",
        Callback = function(Value)
            SelectedFood = ToyList[Value]
        end
    })

    ConfigGroup:AddSlider("AntiInputHoldDuration", {
        Text = "Hold Duration",
        Default = 0.02,
        Min = 0.01,
        Max = 1.0,
        Rounding = 2,
        Callback = function(Value)
            getgenv().AntiInputHoldDuration = Value
        end
    })
end

-- BLOCK 4: ANTI EXPLODE
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local StruggleEvent = E.StruggleEvent
    local RagdollRemote = E.RagdollRemote
    local MainProtection = E.MainProtection
    local dhrp = E.dhrp
    local dhum = E.dhum
    local Toggles = Toggles

    local antiExplodeConn = nil

    MainProtection:AddToggle("AntiExplode", {
        Text = "Anti Explode",
        Default = false,
        Callback = function(v)
            if antiExplodeConn then
                antiExplodeConn:Disconnect()
                antiExplodeConn = nil
            end
            if not v then return end

            antiExplodeConn = WS.ChildAdded:Connect(function(x)
                local r, h = dhrp(), dhum()
                if (Toggles.AntiExplode and safeValue(Toggles.AntiExplode, false)) and r and h and x:IsA("BasePart") and x.Name == "Part" and (x.Position - r.Position).Magnitude < 40 then
                    r.Anchored = true
                    task.wait(0.01)
                    r.Anchored = false
                    r.AssemblyLinearVelocity = Vector3.zero
                    pcall(function() h:ChangeState(Enum.HumanoidStateType.Running) end)
                end
            end)
        end
    })
end

-- BLOCK 5: ANTI BURN / PAINT / VOID / BANANA / TOUCH KILL
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local RS = E.RS
    local RagdollRemote = E.RagdollRemote
    local MainProtection = E.MainProtection
    local dhrp = E.dhrp
    local dhum = E.dhum
    local Toggles = Toggles
    local RunService = game:GetService("RunService")
    local Players = game:GetService("Players")

    local antiBurnActive = false
    local antiBurnConns = {}

    function antiBurnStop()
        for _, c in ipairs(antiBurnConns) do
            pcall(function() c:Disconnect() end)
        end
        antiBurnConns = {}
    end

    function antiBurnWatch(char)
        if not char then return end

        local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
        local hrp = char:FindFirstChild("HumanoidRootPart") or char:WaitForChild("HumanoidRootPart", 5)
        if not hum or not hrp then return end

        local firePart = hrp:FindFirstChild("FirePlayerPart") or char:FindFirstChild("FirePlayerPart", true)
        local canBurnValue = firePart and firePart:FindFirstChild("CanBurn")
        if not canBurnValue then return end

        local extinguishPart
        pcall(function()
            extinguishPart = WS:WaitForChild("Map"):WaitForChild("Hole")
                :WaitForChild("PoisonBigHole"):WaitForChild("ExtinguishPart")
            extinguishPart.Size = Vector3.new(0.5, 0.5, 0.5)
            extinguishPart.Transparency = 1
            local tex = extinguishPart:FindFirstChild("Tex")
            if tex then tex.Transparency = 1 end
        end)

        local function extinguish()
            if not antiBurnActive or not extinguishPart or not firePart then return end

            while antiBurnActive and canBurnValue.Parent and canBurnValue.Value do
                pcall(function()
                    if firetouchinterest then
                        firetouchinterest(firePart, extinguishPart, 0)
                        task.wait()
                        firetouchinterest(firePart, extinguishPart, 1)
                    else
                        extinguishPart.CFrame = firePart.CFrame * CFrame.new(
                            math.random(-1, 1),
                            math.random(-1, 1),
                            math.random(-1, 1)
                        )
                        task.wait()
                        extinguishPart.Position = Vector3.new(0, -100, 0)
                    end
                end)
                task.wait()
            end
        end

        local c = canBurnValue.Changed:Connect(function(propertyName)
            if propertyName and antiBurnActive then
                task.spawn(extinguish)
            end
        end)
        table.insert(antiBurnConns, c)

        if antiBurnActive and canBurnValue.Value then
            task.spawn(extinguish)
        end
    end

    MainProtection:AddToggle("AntiBurn", {
        Text = "Anti Burn",
        Default = false,
        Callback = function(Value)
            antiBurnActive = Value
            antiBurnStop()
            if Value then
                task.spawn(function()
                    antiBurnWatch(P.Character)
                end)
            end
        end
    })

    local antiBurnRespawn = P.CharacterAdded:Connect(function(char)
        if antiBurnActive then
            task.spawn(function()
                antiBurnWatch(char)
            end)
        end
    end)
    table.insert(antiBurnConns, antiBurnRespawn)

    local antiPoisonEnabled = false
    function setAntiPoison(enabled)
        antiPoisonEnabled = enabled
        local map = WS:FindFirstChild("Map")
        local hole = map and map:FindFirstChild("Hole")
        if not hole then return end

        for _, folderName in ipairs({"PoisonBigHole", "PoisonSmallHole"}) do
            local folder = hole:FindFirstChild(folderName)
            if folder then
                for _, obj in ipairs(folder:GetDescendants()) do
                    if obj:IsA("Script") or obj:IsA("LocalScript") then
                        pcall(function() obj.Disabled = enabled end)
                    elseif obj:IsA("BasePart") then
                        pcall(function()
                            obj.CanTouch = not enabled
                            if enabled then obj.CanCollide = false end
                        end)
                    end
                end
            end
        end
    end

    MainProtection:AddToggle("AntiPoison", {
        Text = "Anti Poison",
        Default = false,
        Callback = function(v)
            setAntiPoison(v)
        end
    })

    local AntiStickyActive = false
    local StickyScriptName = "StickyPartsTouchDetection"

    function DisableStickyScripts()
        local playerScripts = LocalPlayer:FindFirstChild("PlayerScripts")
        if not playerScripts then
            return
        end

        for _, script in pairs(playerScripts:GetChildren()) do
            if script.Name == StickyScriptName then
                script.Disabled = true
            end
        end
    end

    function RemoveStickyScripts()
        local playerScripts = LocalPlayer:FindFirstChild("PlayerScripts")
        if not playerScripts then
            return
        end

        for _, script in pairs(playerScripts:GetChildren()) do
            if script.Name == StickyScriptName then
                script:Destroy()
            end
        end
    end

    function RestoreStickyScripts()
        local playerScripts = LocalPlayer:FindFirstChild("PlayerScripts")
        if not playerScripts then
            return
        end

        local sourceScript = ReplicatedStorage:FindFirstChild(StickyScriptName)
        if sourceScript then
            sourceScript:Clone().Parent = playerScripts
        end
    end

    function ToggleAntiSticky(Value)
        AntiStickyActive = Value

        if Value then
            DisableStickyScripts()
            RemoveStickyScripts()
        else
            RestoreStickyScripts()
        end
    end

    LocalPlayer.CharacterAdded:Connect(function()
        if AntiStickyActive then
            task.wait(0.5)
            DisableStickyScripts()
            RemoveStickyScripts()
        end
    end)

    LocalPlayer:WaitForChild("PlayerScripts", 9e9).ChildAdded:Connect(function(child)
        if AntiStickyActive and child.Name == StickyScriptName then
            task.defer(function()
                if child and child.Parent then
                    pcall(function()
                        child.Disabled = true
                        child:Destroy()
                    end)
                end
            end)
        end
    end)

    MainProtection:AddToggle("AntiSticky", {
        Text = "Anti Sticky",
        Default = false,
        Callback = function(Value)
            ToggleAntiSticky(Value)
        end
    })

    local paintPartsBackup = {}
    local paintConnections = {}

    function deleteAllPaintParts()
        for _, obj in ipairs(WS:GetDescendants()) do
            if obj:IsA("BasePart") and obj.Name == "PaintPlayerPart" then
                local clone = obj:Clone()
                clone.Archivable = true
                paintPartsBackup[obj:GetDebugId()] = { clone = clone, parent = obj.Parent }
                obj:Destroy()
            end
        end
    end

    function restorePaintParts()
        for _, data in pairs(paintPartsBackup) do
            if data.clone and data.parent then
                data.clone.Parent = data.parent
            end
        end
        paintPartsBackup = {}
    end

    function disconnectPaintWatchers()
        for _, conn in ipairs(paintConnections) do
            if conn.Connected then conn:Disconnect() end
        end
        paintConnections = {}
    end

    MainProtection:AddToggle("AntiPaint", {
        Text = "Anti Paint",
        Default = false,
        Callback = function(v)
            if v then
                deleteAllPaintParts()
                local conn = WS.DescendantAdded:Connect(function(obj)
                    if obj:IsA("BasePart") and obj.Name == "PaintPlayerPart" then
                        task.defer(function()
                            if obj and obj.Parent then
                                local clone = obj:Clone()
                                clone.Archivable = true
                                paintPartsBackup[obj:GetDebugId()] = { clone = clone, parent = obj.Parent }
                                obj:Destroy()
                            end
                        end)
                    end
                end)
                table.insert(paintConnections, conn)
            else
                restorePaintParts()
                disconnectPaintWatchers()
            end
        end
    })

    local antiBlobEnabled = false
    local antiBlobConnection = nil

    function checkAntiBlob(blob, myHRP, myAttach, humanoid)
        if not blob or not blob.Parent then return end
        local blobScript = blob:FindFirstChild("BlobmanSeatAndOwnerScript")
        if not blobScript or not myAttach then return end

        for _, side in ipairs({"Left", "Right"}) do
            local detector = blob:FindFirstChild(side .. "Detector")
            if detector then
                local weld = detector:FindFirstChild(side .. "Weld")
                local align = detector:FindFirstChild(side .. "AlignOrientation")
                if weld and weld:IsA("AlignPosition") and align and weld.Attachment0 == myAttach then
                    pcall(function()
                        if humanoid then humanoid.PlatformStand = true end
                        if RagdollRemote then RagdollRemote:FireServer(myHRP, 0) end
                        align.Attachment0 = nil
                        weld.Attachment0 = nil
                        weld.Enabled = false
                        align.Enabled = false
                        task.wait()
                        weld.Enabled = true
                        align.Enabled = true
                        if humanoid then humanoid.PlatformStand = false end
                    end)
                end
            end
        end
    end

    function scanAntiBlobFolder(folder, myHRP, myAttach, humanoid)
        if not folder then return end
        for _, blob in ipairs(folder:GetChildren()) do
            if blob.Name == "CreatureBlobman" then
                checkAntiBlob(blob, myHRP, myAttach, humanoid)
            end
        end
    end

    function startAntiBlob()
        if antiBlobConnection then
            antiBlobConnection:Disconnect()
            antiBlobConnection = nil
        end
        antiBlobConnection = RunService.Stepped:Connect(function()
            if not antiBlobEnabled then return end
            local char = P.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local attach = hrp and hrp:FindFirstChild("RootAttachment")
            if not hum or not hrp or not attach then return end

            scanAntiBlobFolder(WS:FindFirstChild(P.Name .. "SpawnedInToys"), hrp, attach, hum)
            for _, player in ipairs(Players:GetPlayers()) do
                if player ~= P then
                    scanAntiBlobFolder(WS:FindFirstChild(player.Name .. "SpawnedInToys"), hrp, attach, hum)
                end
            end

            local plots = WS:FindFirstChild("PlotItems")
            if plots then
                for i = 1, 5 do
                    scanAntiBlobFolder(plots:FindFirstChild("Plot" .. i), hrp, attach, hum)
                end
            end
        end)
    end

    MainProtection:AddToggle("AntiBlob", {
        Text = "Anti Blob",
        Default = false,
        Callback = function(Value)
            antiBlobEnabled = Value and true or false
            if antiBlobConnection then
                antiBlobConnection:Disconnect()
                antiBlobConnection = nil
            end
            if antiBlobEnabled then startAntiBlob() end
        end,
    })

    local originalFallenHeight = WS.FallenPartsDestroyHeight

    MainProtection:AddToggle("AntiVoid", {
        Text = "Anti Void",
        Default = false,
        Callback = function(v)
            if v then
                WS.FallenPartsDestroyHeight = 0/0
            else
                WS.FallenPartsDestroyHeight = originalFallenHeight
            end
        end
    })

    local posLockConnection = nil
    local posLockPosition = nil

    MiscellaneousGroup:AddToggle("PosLock", {
        Text = "Pos Lock",
        Default = false,
        Callback = function(Value)
            if posLockConnection then
                posLockConnection:Disconnect()
                posLockConnection = nil
            end

            if Value then
                local char = P.Character
                local root = char and char:FindFirstChild("HumanoidRootPart")
                if root then posLockPosition = root.CFrame end

                if posLockPosition then
                    posLockConnection = RunService.RenderStepped:Connect(function()
                        local char2 = P.Character
                        local root2 = char2 and char2:FindFirstChild("HumanoidRootPart")
                        if root2 and posLockPosition then
                            pcall(function()
                                local assembly = root2.AssemblyRootPart or root2
                                assembly.AssemblyLinearVelocity = Vector3.zero
                                assembly.AssemblyAngularVelocity = Vector3.zero
                                local offset = assembly.CFrame:ToObjectSpace(root2.CFrame)
                                assembly.CFrame = posLockPosition * offset:Inverse()
                            end)
                        end
                    end)
                end
            else
                pcall(function()
                    local char2 = P.Character
                    local root2 = char2 and char2:FindFirstChild("HumanoidRootPart")
                    if root2 and posLockPosition then
                        root2.CFrame = posLockPosition
                        root2.AssemblyLinearVelocity = Vector3.zero
                        root2.AssemblyAngularVelocity = Vector3.zero
                    end
                end)
                posLockPosition = nil
            end
        end
    })

    local antiBananaThread = nil
    local antiBananaCharConn = nil

    MainProtection:AddToggle("AntiBanana", {
        Text = "Anti Banana",
        Default = false,
        Callback = function(v)
            if antiBananaThread then
                task.cancel(antiBananaThread)
                antiBananaThread = nil
            end
            if antiBananaCharConn then
                antiBananaCharConn:Disconnect()
                antiBananaCharConn = nil
            end
            if not v then return end

            local function resetBananaState(char)
                task.spawn(function()
                    local h = char and char:FindFirstChildOfClass("Humanoid")
                    if not h then return end
                    pcall(function()
                        h.Sit = false
                        h.PlatformStand = false
                        local seat = h.SeatPart
                        if seat then
                            local weld = seat:FindFirstChild("SeatWeld")
                            if weld then weld:Destroy() end
                        end
                        h:ChangeState(Enum.HumanoidStateType.GettingUp)
                        h:ChangeState(Enum.HumanoidStateType.Running)
                    end)
                end)
            end

            antiBananaCharConn = P.CharacterAdded:Connect(function(char)
                if safeValue(Toggles.AntiBanana, false) then
                    task.wait(0.1)
                    resetBananaState(char)
                end
            end)

            antiBananaThread = task.spawn(function()
                while safeValue(Toggles.AntiBanana, false) do
                    local char = P.Character
                    local h = char and char:FindFirstChildOfClass("Humanoid")
                    if h and h.Health > 0 then
                        pcall(function()
                            h.Sit = false
                            h.PlatformStand = false
                            local seat = h.SeatPart
                            if seat then
                                local weld = seat:FindFirstChild("SeatWeld")
                                if weld then weld:Destroy() end
                            end
                            h:ChangeState(Enum.HumanoidStateType.GettingUp)
                            h:ChangeState(Enum.HumanoidStateType.Running)
                        end)
                    end
                    task.wait()
                end
            end)
        end
    })

    -- ANTI TOUCH KILL
    local touchKillActive = false
    local touchKillMainThread = nil
    local touchKillMoveThread = nil

    MainProtection:AddToggle("TouchKill", {
        Text = "Touch Kill",
        Default = false,
        Tooltip = "Kills players you touch",
        Callback = function(state)
            touchKillActive = state

            if state then
                touchKillMainThread = task.spawn(function()
                    while touchKillActive do
                        task.wait(0.1)
                        for _, plr in pairs(Players:GetPlayers()) do
                            if plr ~= P and plr.Character then
                                for _, v in pairs(plr.Character:GetDescendants()) do
                                    if v:IsA("BasePart") then
                                        v.CanCollide = false
                                    end
                                end
                            end
                        end
                    end
                end)

                touchKillMoveThread = task.spawn(function()
                    while touchKillActive do
                        task.wait()
                        local isHeld = P:FindFirstChild("IsHeld")
                        if isHeld and isHeld.Value then continue end
                        local char = P.Character
                        if char and char:FindFirstChild("Humanoid") then
                            char.Humanoid:Move(Vector3.one * 1e31)
                        end
                    end
                end)
            else
                if touchKillMainThread then task.cancel(touchKillMainThread); touchKillMainThread = nil end
                if touchKillMoveThread then task.cancel(touchKillMoveThread); touchKillMoveThread = nil end
            end
        end,
    })
end

-- Anti Ragdoll (On Blob)
do
    local E = _G._EchoDefence
    local P = E.P
    local RS = E.RS
    local MainProtection = E.MainProtection

    local AntiRagBlob = false
    local RagdolledSit = false
    local Cons = {}

    function ApplyAntiRagdoll(char)
        if not char or not AntiRagBlob then return end

        local hum = char:WaitForChild("Humanoid", 5)
        local HRP = char:WaitForChild("HumanoidRootPart", 5)
        if not (hum and HRP) then return end

        if Cons["ARSeat"] then Cons["ARSeat"]:Disconnect() end
        Cons["ARSeat"] = hum:GetPropertyChangedSignal("SeatPart"):Connect(function()
            if hum.SeatPart and hum.SeatPart.Parent and hum.SeatPart.Parent.Name == "CreatureBlobman" and not RagdolledSit then
                RagdolledSit = true
                local Seat = hum.SeatPart
                while not hum.Sit do task.wait() end

                RS.CharacterEvents.RagdollRemote:FireServer(HRP, 3)

                local ragdolledVal = hum:FindFirstChild("Ragdolled")
                while ragdolledVal and not ragdolledVal.Value and not hum.Sit do task.wait() end

                task.wait(0.4)
                hum.Sit = false
                Seat:Sit(hum)

                task.delay(0.25, function()
                    while hum and hum.SeatPart do
                        RS.CharacterEvents.RagdollRemote:FireServer(HRP, 1)
                        task.wait(0.05)
                    end
                    RagdolledSit = false
                end)
            end
        end)
    end

    MainProtection:AddToggle("AntiRagdollOnBlob", {
        Text = "Anti Ragdoll (On Blob)",
        Default = false,
        Callback = function(Value)
            AntiRagBlob = Value
            RagdolledSit = false

            if Cons["ARChar"] then Cons["ARChar"]:Disconnect() end
            if Cons["ARSeat"] then Cons["ARSeat"]:Disconnect() end
            Cons["ARChar"] = nil
            Cons["ARSeat"] = nil

            if AntiRagBlob then
                ApplyAntiRagdoll(P.Character)
                Cons["ARChar"] = P.CharacterAdded:Connect(ApplyAntiRagdoll)
            end
        end
    })
end

-- LOOP TP
local EchoDefence = _G._EchoDefence
local MainProtection = EchoDefence and EchoDefence.MainProtection
local P = EchoDefence and EchoDefence.P or LocalPlayer

if not MainProtection then
    warn("[Echo] Defence Main Protection group was not created.")
end

local loopTPActive = false
local loopTPThread = nil
_G.__EchoLoopTPMode = _G.__EchoLoopTPMode or "Loop TP (Random)"

function stopLoopTP()
    loopTPActive = false
    if loopTPThread then
        pcall(function() task.cancel(loopTPThread) end)
        loopTPThread = nil
    end
    local char = P.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            pcall(function() hum.PlatformStand = false end)
        end
    end
end

function startLoopTP()
    stopLoopTP()
    loopTPActive = true
    loopTPThread = task.spawn(function()
        local angle = 0
        while loopTPActive do
            local char = P.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            local hum = char and char:FindFirstChildOfClass("Humanoid")

            if hrp and hum then
                pcall(function()
                    hum.PlatformStand = true

                    if _G.__EchoLoopTPMode == "Loop TP (OP / Tornado)" then
                        angle = angle + 0.5
                        local rad = math.rad(angle)
                        hrp.CFrame = CFrame.new(math.cos(rad) * 10000, 0, math.sin(rad) * 10000)
                    else
                        hrp.CFrame = CFrame.new(
                            math.random(-500, 500),
                            math.random(30, 480),
                            math.random(-500, 500)
                        )
                    end

                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.AssemblyAngularVelocity = Vector3.zero
                end)
            end

            task.wait(_G.__EchoLoopTPMode == "Loop TP (OP / Tornado)" and 0.03 or 0.05)
        end
    end)
end

MiscellaneousGroup:AddToggle("LoopTP", {
    Text = "Loop TP",
    Default = false,
    Callback = function(Value)
        if Value then
            if _G.__EchoLoopTPMode == "None" then
                stopLoopTP()
                return
            end
            startLoopTP()
        else
            stopLoopTP()
        end
    end,
})

-- ANTI SNOWBALL / LOOP RAGDOLL + DELETE LEGS
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local RagdollRemote = E.RagdollRemote
    local MainProtection = E.MainProtection
    local Toggles = Toggles

    local loopRagdollActive = false
    local loopRagdollThread = nil

    MainProtection:AddToggle("LoopRagdollAntiSnowball", {
        Text = "Anti Snowball / Loop Ragdoll",
        Default = false,
        Callback = function(Value)
            loopRagdollActive = Value

            if loopRagdollThread then
                task.cancel(loopRagdollThread)
                loopRagdollThread = nil
            end

            if not Value then return end

            loopRagdollThread = task.spawn(function()
                while loopRagdollActive do
                    pcall(function()
                        local char = P.Character
                        local hrp = char and char:FindFirstChild("HumanoidRootPart")
                        if hrp and RagdollRemote then
                            RagdollRemote:FireServer(hrp, 0.5)
                        end
                    end)
                    task.wait(0.05)
                end
            end)
        end,
    })

    -- ANTI LAG + AUTO ANTI LAG
    do
        while not game:IsLoaded() do task.wait() end

        local Players2 = game:GetService("Players")
        local LocalPlayer2 = Players2.LocalPlayer
        local ReplicatedStorage2 = game:GetService("ReplicatedStorage")

        local antiLagEnabled = true

        local function EchoStartAntiLag()
            _G.TheWorstAntiLagEnabled = true

            if _G.TheWorstAntiLagConn then
                pcall(function()
                    _G.TheWorstAntiLagConn:Disconnect()
                end)
                _G.TheWorstAntiLagConn = nil
                if not _G.TheWorstAntiLagEnabled then return end
            end

            local CreateGrabLine = ReplicatedStorage2:WaitForChild("GrabEvents", 9e9):WaitForChild("CreateGrabLine", 9e9)
            local BeamMove = LocalPlayer2:WaitForChild("PlayerScripts", 9e9):WaitForChild("CharacterAndBeamMove", 9e9)
            BeamMove.Enabled = false

            task.spawn(function()
                while _G.TheWorstAntiLagEnabled and task.wait(1) do
                    BeamMove.Enabled = true
                end
            end)

            _G.TheWorstAntiLagConn = CreateGrabLine.OnClientEvent:Connect(function(Player, _, CF)
                if Player == LocalPlayer2 then return end

                local PlayerChar = Player ~= LocalPlayer2 and Player.Character
                local PlayerRoot = PlayerChar and PlayerChar:FindFirstChild("HumanoidRootPart")
                if not PlayerRoot or typeof(CF) == "Vector3" or not CF.Position then return end
                if vector.magnitude(PlayerRoot.Position - CF.Position) < 10000 then return end

                BeamMove.Enabled = false

                for _, Plr in Players2:GetPlayers() do
                    if not Plr.Character then continue end
                    local GrabParts = Plr.Character:FindFirstChild("GrabParts")
                    if GrabParts then GrabParts:Destroy() end
                end
            end)
        end

        local function EchoStopAntiLag()
            _G.TheWorstAntiLagEnabled = false

            if _G.TheWorstAntiLagConn then
                pcall(function()
                    _G.TheWorstAntiLagConn:Disconnect()
                end)
                _G.TheWorstAntiLagConn = nil
            end

            pcall(function()
                local BeamMove = LocalPlayer2:WaitForChild("PlayerScripts", 2):FindFirstChild("CharacterAndBeamMove")
                if BeamMove then
                    BeamMove.Enabled = true
                end
            end)
        end

        MainProtection:AddToggle("EchoAntiLag", {
            Text = "Anti Lag",
            Default = true,
            Tooltip = "Protects against abnormal grab-line lag.",
            Callback = function(Value)
                antiLagEnabled = Value and true or false
                if antiLagEnabled then
                    EchoStartAntiLag()
                else
                    EchoStopAntiLag()
                end
            end,
        })

        MainProtection:AddToggle("EchoAutoAntiLag", {
            Text = "Auto Anti Lag",
            Default = true,
            Tooltip = "Automatically enables Anti Lag.",
            Callback = function(Value)
                if Value and Toggles.EchoAntiLag and not Toggles.EchoAntiLag.Value then
                    pcall(function()
                        Toggles.EchoAntiLag:SetValue(true)
                    end)
                end
            end,
        })

        EchoStartAntiLag()

        if Toggles.EchoAutoAntiLag and Toggles.EchoAutoAntiLag.Value then
            task.defer(function()
                if Toggles.EchoAutoAntiLag and Toggles.EchoAutoAntiLag.Value and Toggles.EchoAntiLag and not Toggles.EchoAntiLag.Value then
                    pcall(function()
                        Toggles.EchoAntiLag:SetValue(true)
                    end)
                end
            end)
        end

        pcall(function()
            Library:OnUnload(function()
                EchoStopAntiLag()
            end)
        end)
    end

    MainProtection:AddButton({
        Text = "Delete Legs",
        Func = function()
            local char = P.Character
            if not char then
                char = P.CharacterAdded:Wait()
            end

            local leftLeg = char:FindFirstChild("Left Leg")
            local rightLeg = char:FindFirstChild("Right Leg")
            local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
            local hrp = char:FindFirstChild("HumanoidRootPart")

            if leftLeg and rightLeg and torso and hrp then
                local originalFallHeight = WS.FallenPartsDestroyHeight
                local originalCFrame = torso.CFrame

                WS.FallenPartsDestroyHeight = -100
                RagdollRemote:FireServer(hrp, 2)

                task.wait(0.5)
                leftLeg.CFrame = CFrame.new(0, -10000, 0)
                rightLeg.CFrame = CFrame.new(0, -10000, 0)

                task.wait(0.3)
                torso.CFrame = CFrame.new(0, -9970, 0)

                task.wait(0.5)
                torso.CFrame = originalCFrame

                task.wait(0.5)
                WS.FallenPartsDestroyHeight = originalFallHeight

                Library:Notify({
                        Description = "Legs deleted successfully!",
                    Time = 2,
                })
            else
                Library:Notify({
                        Description = "Could not find legs or torso!",
                    Time = 2,
                })
            end
        end,
    })
end

-- ANTI KICK
do
    local E = _G._EchoDefence
    local P = E.P
    local WS = E.WS
    local RS = E.RS
    local SetNetOwner = E.SetNetOwner
    local StruggleEvent = E.StruggleEvent
    local RagdollRemote = E.RagdollRemote
    local SpawnToyRemote = E.SpawnToyRemote
    local DestroyToy = E.DestroyToy
    local StickyEvent = E.StickyEvent
    local dhrp = E.dhrp
    local dhum = E.dhum
    local checkNetworkOwner = E.checkNetworkOwner
    local Toggles = Toggles
    local Players = game:GetService("Players")
    local Debris = game:GetService("Debris")
    local RunService = game:GetService("RunService")
    local Stats = game:GetService("Stats")

    local AntiKickGroup = Tabs.Defence:AddLeftGroupbox("Anti Kick", "shield-alert")

    -- Anti Kick stick settings
    local StickToyMethod = "NinjaShuriken"
    local StickTargetPart = "Torso"
    local StickOffsetX, StickOffsetY, StickOffsetZ = 0, 0, 0
    local StickRotationX, StickRotationY, StickRotationZ = 0, 0, 0

    local AntiKickToyValues = {
        "NinjaShuriken",
        "NinjaKunai",
        "ToolCleaver",
        "ToolDiggingForkRusty",
        "ToolPencil",
        "ToolPickaxe",
        "NinjaKatana",
    }

    local AntiKickToyImages = {
        ["NinjaShuriken"] = "rbxassetid://140368128539182",
        ["NinjaKunai"] = "rbxassetid://85197427890853",
        ["ToolCleaver"] = "rbxassetid://70617572473301",
        ["ToolDiggingForkRusty"] = "rbxassetid://99651991375781",
        ["ToolPencil"] = "rbxassetid://92908917595707",
        ["ToolPickaxe"] = "rbxassetid://129936050231156",
        -- Roblox catalog Katana thumbnail used as the fallback UI image.
        ["NinjaKatana"] = "rbxthumb://type=Asset&id=14932219587&w=150&h=150",
    }

    local AntiKickToyNames = {
        ["NinjaShuriken"] = "Shuriken",
        ["NinjaKunai"] = "Kunai",
        ["ToolCleaver"] = "Cleaver",
        ["ToolDiggingForkRusty"] = "Digging Fork",
        ["ToolPencil"] = "Pencil",
        ["ToolPickaxe"] = "Pickaxe",
        ["NinjaKatana"] = "Katana",
    }

    local AntiKickTabbox = AntiKickGroup:AddTabbox()
    local AntiKickTogglesTab = AntiKickTabbox:AddTab("Toggles", "scan-check")
    local StickSettingsTab = AntiKickTabbox:AddTab("Settings", "settings")

    local pcldCharacterConn = nil
    AntiKickTogglesTab:AddToggle("BreakPCLD", {
        Text = "Break PCLD",
        Default = false,
        Callback = function(Value)
            local hkExpectDeath = false
            local hkSalmonList = {}

            if pcldCharacterConn then
                pcldCharacterConn:Disconnect()
                pcldCharacterConn = nil
            end

            if Value then
                hkSalmonList[LocalPlayer.UserId] = true

                local function hkApplySalmon(char)
                    if not char then return end
                    local newHum = char:WaitForChild("Humanoid", 5)
                    if not newHum then return end

                    if hkSalmonList[LocalPlayer.UserId] and not hkExpectDeath then
                        hkExpectDeath = true
                        newHum:ChangeState(Enum.HumanoidStateType.Dead)
                    else
                        hkExpectDeath = false
                    end
                end

                pcldCharacterConn = LocalPlayer.CharacterAdded:Connect(function(char)
                    hkApplySalmon(char)
                end)

                hkExpectDeath = false
                local char = LocalPlayer.Character
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hum then
                    hum.Health = 0
                end
            else
                hkSalmonList[LocalPlayer.UserId] = nil
                hkExpectDeath = false
            end
        end
    })

    local autoResetConnection = nil
    local autoResetBusy = false

    local function disconnectAutoResetConnections()
        if autoResetConnection then
            pcall(function() autoResetConnection:Disconnect() end)
            autoResetConnection = nil
        end
    end

    local function echoResetCharacter()
        if autoResetBusy then return end
        autoResetBusy = true

        pcall(function()
            pcall(function()
                _G.__EchoNotificationSound.SoundId = "rbxassetid://97643101798871"
                _G.__EchoNotificationSound:Stop()
                _G.__EchoNotificationSound.TimePosition = 0
                _G.__EchoNotificationSound:Play()
            end)
           _G.__EchoNotify("Anti Cheat Warning", "anticheat_warn", 4, "AntiCheatWarn")
        end)

        pcall(function()
            local char = P.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum then
                hum.Health = 0
            end
        end)

        task.delay(1, function()
            autoResetBusy = false
        end)
    end

    local function startAutoReset()
        disconnectAutoResetConnections()

        local correctionEvents = ReplicatedStorage:FindFirstChild("GameCorrectionEvents")
        if not correctionEvents then
            correctionEvents = ReplicatedStorage:WaitForChild("GameCorrectionEvents", 9)
        end
        local correctionNotify = correctionEvents and correctionEvents:FindFirstChild("GameCorrectionsNotify")
        if not correctionNotify and correctionEvents then
            correctionNotify = correctionEvents:WaitForChild("GameCorrectionsNotify", 9)
        end
        if not correctionNotify then
            return
        end

        autoResetConnection = correctionNotify.OnClientEvent:Connect(function(reason)
            if reason == "Flying" then
                echoResetCharacter()
            end
        end)
    end

    local function stopAutoReset()
        disconnectAutoResetConnections()
        autoResetBusy = false
    end

    AntiKickTogglesTab:AddToggle("AutoReset", {
        Text = "Auto Reset",
        Default = false,
        Callback = function(Value)
            if Value then
                startAutoReset()
            else
                stopAutoReset()
            end
        end,
    })

    AntiKickTogglesTab:AddToggle("AutoLeave", {
        Text = "Auto Leave",
        Default = false,
        Callback = function(Value)
            if not Value then return end
            task.spawn(function()
                while safeValue(Toggles.AutoLeave, false) do
                    task.wait(0.25)
                    pcall(function()
                        local char = P.Character
                        local hum = char and char:FindFirstChildOfClass("Humanoid")
                        if hum and hum.Health <= 0 then
                            TeleportService:Teleport(game.PlaceId, P)
                        end
                    end)
                end
            end)
        end,
    })

    StickSettingsTab:AddDropdown("AntiKickToyMethod", {
        Text = "Toy Method Select",
        Values = AntiKickToyValues,
        Default = StickToyMethod,
        Multi = false,
        Callback = function(Value)
            if AntiKickToyNames[Value] then
                StickToyMethod = Value
            end
        end,
        FormatDisplayValue = function(Value)
            return AntiKickToyNames[Value] or Value
        end,
    })

    StickSettingsTab:AddDropdown("AntiKickTargetPart", {
        Text = "Stick Target Part",
        Values = {"HoldPart", "FirePlayerPart", "HumanoidRootPart", "Head", "Torso"},
        Default = StickTargetPart,
        Multi = false,
        Callback = function(Value)
            StickTargetPart = Value
        end,
    })

    -- ── XYZ Offset: continuous sliders (replaces typed inputs) ──────
    StickSettingsTab:AddSlider("AntiKickOffsetX", {
        Text = "Stick Offset - X",
        Default = 0,
        Min = -10,
        Max = 10,
        Rounding = 2,
        Suffix = " st",
        Callback = function(Value)
            StickOffsetX = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })
    StickSettingsTab:AddSlider("AntiKickOffsetY", {
        Text = "Stick Offset - Y",
        Default = 0,
        Min = -10,
        Max = 10,
        Rounding = 2,
        Suffix = " st",
        Callback = function(Value)
            StickOffsetY = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })
    StickSettingsTab:AddSlider("AntiKickOffsetZ", {
        Text = "Stick Offset - Z",
        Default = 0,
        Min = -10,
        Max = 10,
        Rounding = 2,
        Suffix = " st",
        Callback = function(Value)
            StickOffsetZ = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })

    StickSettingsTab:AddInput("AntiKickRotationX", {
        Text = "Stick Rotation - X",
        Default = "0",
        Numeric = true,
        Finished = true,
        Callback = function(Value)
            StickRotationX = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })
    StickSettingsTab:AddInput("AntiKickRotationY", {
        Text = "Stick Rotation - Y",
        Default = "0",
        Numeric = true,
        Finished = true,
        Callback = function(Value)
            StickRotationY = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })
    StickSettingsTab:AddInput("AntiKickRotationZ", {
        Text = "Stick Rotation - Z",
        Default = "0",
        Numeric = true,
        Finished = true,
        Callback = function(Value)
            StickRotationZ = tonumber(Value) or 0
            if AntiKickApplyPosition then AntiKickApplyPosition() end
        end,
    })

    -- Named Anti Kick presets. There is no fixed slot limit.
    local AntiKickPresets = {}
    local AntiKickCurrentPresetName = "My Preset"
    local AntiKickPresetValues = {}
    StickSettingsTab:AddInput("AntiKickPresetName", {
        Text = "Preset Name",
        Default = AntiKickCurrentPresetName,
        Placeholder = "Enter any preset name",
        Finished = true,
        Callback = function(Value)
            local name = tostring(Value or "")
            if name:match("%S") then
                AntiKickCurrentPresetName = name
            end
        end,
    })

    local AntiKickPresetDropdown = StickSettingsTab:AddDropdown("AntiKickPresetSelect", {
        Text = "Saved Presets",
        Values = AntiKickPresetValues,
        Default = nil,
        AllowNull = true,
        Multi = false,
        Callback = function(Value)
            if Value and tostring(Value) ~= "" then
                AntiKickCurrentPresetName = tostring(Value)
                if Options.AntiKickPresetName then
                    pcall(function() Options.AntiKickPresetName:SetValue(AntiKickCurrentPresetName) end)
                end
            end
        end,
    })

    function refreshAntiKickPresetDropdown(selectName)
        local names = {}
        for name in pairs(AntiKickPresets) do
            table.insert(names, name)
        end
        table.sort(names, function(a, b)
            return tostring(a):lower() < tostring(b):lower()
        end)
        AntiKickPresetValues = names
        pcall(function()
            AntiKickPresetDropdown:SetValues(names)
        end)
        if selectName and AntiKickPresets[selectName] then
            pcall(function() AntiKickPresetDropdown:SetValue(selectName) end)
        end
    end

    function getAntiKickSettings()
        return {
            TargetPart = StickTargetPart,
            OffsetX = StickOffsetX,
            OffsetY = StickOffsetY,
            OffsetZ = StickOffsetZ,
            RotationX = StickRotationX,
            RotationY = StickRotationY,
            RotationZ = StickRotationZ,
        }
    end

    function applyAntiKickSettings(data)
        if not data then return false end
        StickTargetPart = data.TargetPart or StickTargetPart
        StickOffsetX = tonumber(data.OffsetX) or 0
        StickOffsetY = tonumber(data.OffsetY) or 0
        StickOffsetZ = tonumber(data.OffsetZ) or 0
        StickRotationX = tonumber(data.RotationX) or 0
        StickRotationY = tonumber(data.RotationY) or 0
        StickRotationZ = tonumber(data.RotationZ) or 0

        -- Sliders accept numbers; rotation inputs still accept strings.
        local sliderValues = {
            AntiKickOffsetX = StickOffsetX,
            AntiKickOffsetY = StickOffsetY,
            AntiKickOffsetZ = StickOffsetZ,
        }
        local inputValues = {
            AntiKickRotationX = tostring(StickRotationX),
            AntiKickRotationY = tostring(StickRotationY),
            AntiKickRotationZ = tostring(StickRotationZ),
        }
        for optionName, value in pairs(sliderValues) do
            local option = Options[optionName]
            if option then
                pcall(function() option:SetValue(value) end)
            end
        end
        for optionName, value in pairs(inputValues) do
            local option = Options[optionName]
            if option then
                pcall(function() option:SetValue(value) end)
            end
        end
        if AntiKickApplyPosition then
            task.defer(AntiKickApplyPosition)
        end
        return true
    end

    StickSettingsTab:AddButton({
        Text = "Save Preset",
        Func = function()
            local name = tostring(AntiKickCurrentPresetName or ""):gsub("^%s+", ""):gsub("%s+$", "")
            if name == "" then
                Library:Notify({
                        Description = "Enter a preset name first",
                    Time = 2,
                })
                return
            end

            if AntiKickPresets[name] then
                Library:Notify({
                        Description = name .. " already exists. Use Overwrite.",
                    Time = 2,
                })
                return
            end

            AntiKickPresets[name] = getAntiKickSettings()
            refreshAntiKickPresetDropdown(name)
            Library:Notify({
                Description = name .. " saved",
                Time = 2,
            })
        end,
    })

    StickSettingsTab:AddButton({
        Text = "Overwrite Preset",
        Func = function()
            local name = tostring(AntiKickCurrentPresetName or ""):gsub("^%s+", ""):gsub("%s+$", "")
            if name == "" then
                Library:Notify({
                        Description = "Enter a preset name first",
                    Time = 2,
                })
                return
            end
            if not AntiKickPresets[name] then
                Library:Notify({
                        Description = name .. " does not exist. Save it first.",
                    Time = 2,
                })
                return
            end

            AntiKickPresets[name] = getAntiKickSettings()
            refreshAntiKickPresetDropdown(name)
            Library:Notify({
                Description = name .. " overwritten",
                Time = 2,
            })
        end,
    })

    StickSettingsTab:AddButton({
        Text = "Load Preset",
        Func = function()
            local name = tostring(AntiKickCurrentPresetName or ""):gsub("^%s+", ""):gsub("%s+$", "")
            if name == "" then
                Library:Notify({
                        Description = "Enter a preset name first",
                    Time = 2,
                })
                return
            end

            local data = AntiKickPresets[name]
            if not data then
                Library:Notify({
                        Description = name .. " is empty",
                    Time = 2,
                })
                return
            end

            applyAntiKickSettings(data)
            Library:Notify({
                Description = name .. " loaded",
                Time = 2,
            })
        end,
    })

    local antiKickActive = false
    local antiKickToken = 0

    function antiKickGetOwnFolder()
        return WS:FindFirstChild(P.Name .. "SpawnedInToys")
    end

    function antiKickDestroyToy(toy)
        if toy and toy.Parent and DestroyToy then
            pcall(function()
                DestroyToy:FireServer(toy)
            end)
        end
    end

    function antiKickFindOwnedPlotFolder()
        local plotItems = WS:FindFirstChild("PlotItems")
        local plots = WS:FindFirstChild("Plots")
        local playersInPlots = plotItems and plotItems:FindFirstChild("PlayersInPlots")
        if not plotItems or not plots or not playersInPlots or not playersInPlots:FindFirstChild(P.Name) then
            return nil
        end

        for _, plot in ipairs(plots:GetChildren()) do
            local sign = plot:FindFirstChild("PlotSign")
            local owners = sign and sign:FindFirstChild("ThisPlotsOwners")
            if owners then
                for _, ownerValue in ipairs(owners:GetChildren()) do
                    if ownerValue.Value == P.Name then
                        return plotItems:FindFirstChild(plot.Name)
                    end
                end
            end
        end
        return nil
    end

    function antiKickGetToyFolders()
        local folders = {}
        local seen = {}

        local function addFolder(folder)
            if folder and not seen[folder] then
                seen[folder] = true
                table.insert(folders, folder)
            end
        end

        -- Some servers put spawned toys in the player folder; plot owners can
        -- instead be placed inside PlotItems/<PlotName>. Search both.
        addFolder(antiKickGetOwnFolder())
        addFolder(antiKickFindOwnedPlotFolder())

        return folders
    end

    function antiKickFindCurrentToy()
        for _, folder in ipairs(antiKickGetToyFolders()) do
            local toy = folder:FindFirstChild("AntiKick")
            if toy then
                return toy
            end
        end
        return nil
    end

    function antiKickDestroySupported()
        for _, folder in ipairs(antiKickGetToyFolders()) do
            for _, toy in ipairs(folder:GetChildren()) do
                if AntiKickToyNames[toy.Name] or toy.Name == "AntiKick" then
                    antiKickDestroyToy(toy)
                end
            end
        end
    end

    function antiKickFindTargetPart(character)
        if not character then return nil end

        local part
        if StickTargetPart == "Torso" then
            -- UI says Torso; use the game's actual torso attachment anchor.
            part = character:FindFirstChild("FirePlayerPart", true)
                or character:FindFirstChild("Torso")
                or character:FindFirstChild("UpperTorso")
        elseif StickTargetPart == "FirePlayerPart" then
            part = character:FindFirstChild("FirePlayerPart", true)
        elseif StickTargetPart == "HoldPart" then
            part = character:FindFirstChild("HoldPart", true)
        elseif StickTargetPart == "Head" then
            part = character:FindFirstChild("Head", true)
        elseif StickTargetPart == "HumanoidRootPart" then
            part = character:FindFirstChild("HumanoidRootPart")
        end

        if part and part:IsA("BasePart") then
            return part
        end

        return character:FindFirstChild("Torso")
            or character:FindFirstChild("UpperTorso")
            or character:FindFirstChild("FirePlayerPart", true)
            or character:FindFirstChild("HumanoidRootPart")
    end

    function antiKickIsProperlyStuck(toy)
        if not toy or not toy.Parent then return false end
        local stickyPart = toy:FindFirstChild("StickyPart", true)
        if not stickyPart then return false end
        local weld = stickyPart:FindFirstChild("StickyWeld")
        if not weld or not weld.Part1 then return false end

        local character = P.Character
        if not character then return false end
        local targetPart = antiKickFindTargetPart(character)
        return targetPart ~= nil and weld.Part1 == targetPart
    end

    EchoApplyAntiKickToyTransparency = function(toy)
        if not toy or not toy.Parent or toy.Name ~= "AntiKick" then
            return
        end

        local transparency = math.clamp(tonumber(EchoAntiKickToyTransparency) or 0, 0, 1)
        for _, obj in ipairs(toy:GetDescendants()) do
            if obj:IsA("BasePart") then
                obj.Transparency = transparency
            elseif obj:IsA("Decal") or obj:IsA("Texture") then
                obj.Transparency = transparency
            end
        end
    end

    function antiKickAttach(toy)
        if not toy or not toy.Parent then return false end

        local stickyPart = toy:FindFirstChild("StickyPart", true)
        if not stickyPart or not stickyPart:IsA("BasePart") then
            return false
        end

        local character = P.Character
        local hum = character and character:FindFirstChildOfClass("Humanoid")
        if not character or not hum or hum.Health <= 0 then
            return false
        end

        -- The working source attaches specifically to FirePlayerPart. Keep the
        -- UI setting named Torso, but resolve it to that exact game anchor.
        local targetPart = antiKickFindTargetPart(character)
        if not targetPart or targetPart.Name ~= "FirePlayerPart" then
            targetPart = character:FindFirstChild("FirePlayerPart", true)
        end
        if not targetPart or not targetPart:IsA("BasePart") then
            return false
        end

        -- Match the working source exactly: request ownership only for the
        -- toy's SoundPart. Taking ownership of StickyPart itself can prevent
        -- the server-side sticky weld from being created.
        local soundPart = toy:FindFirstChild("SoundPart", true)
        if soundPart and soundPart:IsA("BasePart") and SetNetOwner then
            pcall(function()
                if not soundPart:FindFirstChild("PartOwner")
                    or soundPart.PartOwner.Value ~= P.Name then
                    SetNetOwner:FireServer(soundPart, soundPart.CFrame)
                end
            end)
        end

        -- Exact working-source base orientation is 0, 90, 90. The three UI
        -- rotation inputs are additive adjustments around that base.
        local attachCF = CFrame.new(
            tonumber(StickOffsetX) or 0,
            tonumber(StickOffsetY) or 0,
            tonumber(StickOffsetZ) or 0
        ) * CFrame.Angles(
            math.rad(tonumber(StickRotationX) or 0),
            math.rad(90 + (tonumber(StickRotationY) or 0)),
            math.rad(90 + (tonumber(StickRotationZ) or 0))
        )

        if not StickyEvent then
            return false
        end

        local fired = pcall(function()
            StickyEvent:FireServer(stickyPart, targetPart, attachCF)
        end)
        if not fired then
            return false
        end

        -- Give the server a moment to create StickyWeld before altering the
        -- toy's local physics state.
        for _ = 1, 10 do
            task.wait(0.03)
            local weld = stickyPart:FindFirstChild("StickyWeld")
            if weld and weld.Part1 == targetPart then
                for _, obj in ipairs(toy:GetDescendants()) do
                    if obj:IsA("Highlight") or obj:IsA("SelectionBox") or obj:IsA("BillboardGui") then
                        obj:Destroy()
                    elseif obj:IsA("BasePart") then
                        obj.CanTouch = false
                        obj.CanCollide = false
                        obj.CanQuery = false
                        obj.CastShadow = false
                    end
                end
                EchoApplyAntiKickToyTransparency(toy)
                if EchoAntiKickToyTransparency >= 1 then
                    for _, obj in ipairs(toy:GetDescendants()) do
                        if obj:IsA("BasePart") then
                            obj.Transparency = 1
                        elseif obj:IsA("Decal") or obj:IsA("Texture") then
                            obj.Transparency = 1
                        end
                    end
                end
                return true
            end
        end

        return false
    end

    function antiKickAttachGuaranteed(toy, token)
        if not toy then return false end

        for _ = 1, 8 do
            if not antiKickActive or token ~= antiKickToken or not toy.Parent then
                return false
            end

            antiKickAttach(toy)
            task.wait(0.4)

            if antiKickIsProperlyStuck(toy) then
                return true
            end
        end

        antiKickAttach(toy)
        task.wait(0.6)
        return antiKickIsProperlyStuck(toy)
    end

    AntiKickApplyPosition = function()
        local toy = EchoAntiKickCurrentToy
        if not antiKickActive or not toy or not toy.Parent then
            return
        end

        -- Re-apply the live offset immediately and for a few frames so the
        -- server-side sticky weld catches the new placement without requiring
        -- a respawn/reset. The current XYZ values are read on every pass.
        local applyToken = antiKickToken
        task.spawn(function()
            for _ = 1, 8 do
                if not antiKickActive or applyToken ~= antiKickToken or not toy.Parent then
                    break
                end
                pcall(function()
                    antiKickAttach(toy)
                end)
                task.wait(0.02)
            end
        end)
    end

    function antiKickSpawnSelected(token)
        local canSpawn = P:FindFirstChild("CanSpawnToy")
        local root = P.Character and P.Character:FindFirstChild("HumanoidRootPart")
        if not canSpawn or not root or not SpawnToyRemote then
            return nil
        end

        local startWait = tick()
        while antiKickActive and token == antiKickToken and not canSpawn.Value do
            if tick() - startWait > 5 then
                return nil
            end
            canSpawn.Changed:Wait()
        end

        if not antiKickActive or token ~= antiKickToken then
            return nil
        end

        local spawnCF = root.CFrame * CFrame.new(0, 12, 20)
        local folders = antiKickGetToyFolders()
        if #folders == 0 then
            return nil
        end

        local spawned = nil
        local childConns = {}
        for _, folder in ipairs(folders) do
            local conn = folder.ChildAdded:Connect(function(child)
                if child.Name == StickToyMethod then
                    spawned = child
                end
            end)
            table.insert(childConns, conn)
        end

        pcall(function()
            SpawnToyRemote:InvokeServer(StickToyMethod, spawnCF, Vector3.zero)
        end)

        local deadline = tick() + 2.5
        repeat
            if spawned and spawned.Parent then
                break
            end
            for _, folder in ipairs(folders) do
                spawned = folder:FindFirstChild(StickToyMethod)
                if spawned then break end
            end
            if spawned then break end
            task.wait(0.03)
        until tick() >= deadline or not antiKickActive or token ~= antiKickToken

        for _, conn in ipairs(childConns) do
            pcall(function() conn:Disconnect() end)
        end

        if spawned and spawned.Parent then
            spawned.Name = "AntiKick"
            return spawned
        end

        return nil
    end

    AntiKickTogglesTab:AddToggle("AntiKickShuriken", {
        Text = "Anti Kick",
        Default = false,
        Callback = function(Value)
            antiKickActive = Value
            antiKickToken = antiKickToken + 1
            local token = antiKickToken
            EchoAntiKickCurrentToy = nil

            if not Value then
                antiKickDestroySupported()
                return
            end

            task.spawn(function()
                local currentToy = nil
                local currentMethod = nil
                local antiKickWasAttached = false
                local antiKickDetachNotifyAt = 0

                while antiKickActive and token == antiKickToken do
                    task.wait(0.05)

                    if currentMethod ~= StickToyMethod then
                        if currentToy and currentToy.Parent then
                            antiKickDestroyToy(currentToy)
                        end
                        currentToy = nil
                        EchoAntiKickCurrentToy = nil
                        currentMethod = StickToyMethod
                    end

                    local character = P.Character
                    local hum = character and character:FindFirstChildOfClass("Humanoid")
                    local root = character and character:FindFirstChild("HumanoidRootPart")
                    if not hum or hum.Health <= 0 or not root then
                        currentToy = nil
                        EchoAntiKickCurrentToy = nil
                        task.wait(0.25)
                        continue
                    end

                    if currentToy and (not currentToy.Parent or currentToy.Name ~= "AntiKick") then
                        if antiKickWasAttached and tick() >= antiKickDetachNotifyAt then
                            _G.__EchoNotify("You're Anti kick has been removed", "antikick_removed", 4, "AntiKickRemoved")
                            antiKickDetachNotifyAt = tick() + 2
                        end
                        antiKickWasAttached = false
                        currentToy = nil
                        EchoAntiKickCurrentToy = nil
                    end

                    if currentToy and antiKickWasAttached and not antiKickIsProperlyStuck(currentToy) then
                        if tick() >= antiKickDetachNotifyAt then
                            _G.__EchoNotify("You're Anti kick has been removed", "antikick_removed", 4)
                            antiKickDetachNotifyAt = tick() + 2
                        end
                        antiKickWasAttached = false
                    end

                    if not currentToy then
                        currentToy = antiKickFindCurrentToy()
                        if currentToy and currentToy.Name == "AntiKick" then
                            local selectedSticky = currentToy:FindFirstChild("StickyPart", true)
                            if not selectedSticky then
                                currentToy = nil
                            end
                        end
                    end

                    if not currentToy then
                        antiKickDestroySupported()
                        currentToy = antiKickSpawnSelected(token)
                    end

                    if currentToy then
                        EchoAntiKickCurrentToy = currentToy

                        if not antiKickAttachGuaranteed(currentToy, token) then
                            antiKickDestroyToy(currentToy)
                            currentToy = nil
                            EchoAntiKickCurrentToy = nil
                            antiKickWasAttached = false
                        else
                            antiKickWasAttached = true
                            local stickyPart = currentToy:FindFirstChild("StickyPart", true)
                            local distance = stickyPart and (root.Position - stickyPart.Position).Magnitude or math.huge
                            if not stickyPart or distance >= 25 then
                                antiKickDestroyToy(currentToy)
                                currentToy = nil
                                EchoAntiKickCurrentToy = nil
                                antiKickWasAttached = false
                            end
                        end
                    end
                end
            end)
        end,
    })

    local CounterGroup = Tabs.Defence:AddRightGroupbox("Counters", "sword")

    local counterAttacksEnabled = false
    local counterMode = "Fling"
    local counterConn = nil

    function getAttacker()
        local char = P.Character
        if not char or not char:FindFirstChild("Head") then return end
        local owner = char.Head:FindFirstChild("PartOwner")
        if not owner or not owner:IsA("StringValue") then return end
        local ok, name = pcall(function() return owner.Value end)
        return ok and name and Players:FindFirstChild(name) or nil
    end

    function performFling(attacker)
        if not attacker or not attacker.Character then return end
        local root = attacker.Character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        pcall(function()
            SetNetOwner:FireServer(root, root.CFrame)
            DestroyToy:FireServer(root)
            local away = (root.Position - P.Character.HumanoidRootPart.Position).Unit
            away = Vector3.new(away.X, 0, away.Z) * 999999999
            local bv = Instance.new("BodyVelocity")
            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
            bv.Velocity = away
            bv.P = 12500
            bv.Parent = root
            Debris:AddItem(bv, 0.1)
        end)
    end

    function performKill(attacker)
        if not attacker or not attacker.Character then return end
        local root = attacker.Character:FindFirstChild("HumanoidRootPart")
        local hum = attacker.Character:FindFirstChild("Humanoid")
        if not root or not hum then return end
        pcall(function()
            SetNetOwner:FireServer(root, root.CFrame)
            DestroyToy:FireServer(root)
            hum:ChangeState(Enum.HumanoidStateType.Dead)
            hum.Health = 0
            attacker.Character:BreakJoints()
        end)
    end

    function performHeaven(attacker)
        if not attacker or not attacker.Character then return end
        local root = attacker.Character:FindFirstChild("HumanoidRootPart")
        if not root then return end
        pcall(function()
            SetNetOwner:FireServer(root, root.CFrame)
            DestroyToy:FireServer(root)
            root.CFrame = CFrame.new(0, 9999999999, 0)
            local bv = Instance.new("BodyVelocity")
            bv.MaxForce = Vector3.new(0, math.huge, 0)
            bv.Velocity = Vector3.new(0, 999999999999, 0)
            bv.P = 12500
            bv.Parent = root
            Debris:AddItem(bv, 5)
        end)
    end

    CounterGroup:AddToggle("CounterAttacks", {
        Text = "Counter Attacks",
        Default = false,
        Callback = function(Value)
            counterAttacksEnabled = Value
            if counterConn then
                counterConn:Disconnect()
                counterConn = nil
            end
            if Value then
                counterConn = RunService.Heartbeat:Connect(function()
                    if not counterAttacksEnabled then return end
                    local attacker = getAttacker()
                    if not attacker then return end
                    if counterMode == "Fling" then performFling(attacker)
                    elseif counterMode == "Kill" then performKill(attacker)
                    elseif counterMode == "Send to Heaven" then performHeaven(attacker) end
                end)
            end
        end
    })

    CounterGroup:AddDropdown("CounterMode", {
        Text = "Counter Mode",
        Values = {"Fling", "Kill", "Send to Heaven"},
        Default = "Fling",
        Callback = function(value) counterMode = value end
    })

    -- ══════════════════════════════════════════════════════════════
    -- FAST AUTO-ATTACH ON SPAWN + ANCESTRY RE-ATTACH WATCHER
    -- Instantly attaches the sticky toy inside the torso after every
    -- respawn.  If anything destroys the toy (anti-kick scan, server
    -- script, another player), the AncestryChanged watcher fires and
    -- re-spawns + re-attaches it within one server frame.
    -- ══════════════════════════════════════════════════════════════

    -- Tracks the AncestryChanged connection so we can clean it up.
    local antiKickRespawnWatchConn = nil

    local function antiKickStopRespawnWatch()
        if antiKickRespawnWatchConn then
            pcall(function() antiKickRespawnWatchConn:Disconnect() end)
            antiKickRespawnWatchConn = nil
        end
    end

    -- Called after every spawn.  Waits for the character to be ready,
    -- then immediately spawns and attaches the sticky toy, and hooks
    -- AncestryChanged so removal triggers an instant re-attach.
    local function antiKickFastAttachOnSpawn(character)
        if not antiKickActive then return end

        -- Wait for essential parts (handles both R6 and R15 load times).
        local hum = character:WaitForChild("Humanoid", 8)
        local hrp = character:WaitForChild("HumanoidRootPart", 8)
        if not hum or not hrp then return end

        -- Bump the token so any prior loop iteration exits cleanly.
        antiKickToken = antiKickToken + 1
        local token = antiKickToken
        EchoAntiKickCurrentToy = nil
        antiKickDestroySupported()

        -- Inner helper: spawns toy, attaches it, and hooks the watcher.
        local function spawnAndAttach()
            if not antiKickActive or token ~= antiKickToken then return end
            antiKickStopRespawnWatch()

            local toy = antiKickSpawnSelected(token)
            if not toy or not toy.Parent then return end

            EchoAntiKickCurrentToy = toy
            local attached = antiKickAttachGuaranteed(toy, token)

            if not attached then
                antiKickDestroyToy(toy)
                EchoAntiKickCurrentToy = nil
                return
            end

            -- Watch for external removal and immediately re-attach.
            antiKickRespawnWatchConn = toy.AncestryChanged:Connect(function(_, newParent)
                if newParent ~= nil then return end          -- still alive
                if not antiKickActive then return end        -- feature off
                antiKickStopRespawnWatch()
                EchoAntiKickCurrentToy = nil
            _G.__EchoNotify("Anti Kick toy removed — re-attaching…", "antikick_reattach", 3, "AntiKickRemoved")
                -- Defer so the server has one frame to settle.
                task.defer(function()
                    if not antiKickActive or token ~= antiKickToken then return end
                    local char2 = P.Character
                    if not char2 then return end
                    spawnAndAttach()
                end)
            end)
        end

        spawnAndAttach()
    end

    -- Re-hook on every respawn.
    P.CharacterAdded:Connect(function(char)
        task.defer(antiKickFastAttachOnSpawn, char)
    end)

    -- Also attach immediately if Anti Kick is already active when the
    -- current character is alive (covers mid-session enable).
    if antiKickActive and P.Character then
        task.defer(antiKickFastAttachOnSpawn, P.Character)
    end

    pcall(function()
        Library:OnUnload(function()
            antiKickStopRespawnWatch()

            if antiInputThread then task.cancel(antiInputThread) end
            if loopTPThread then task.cancel(loopTPThread) end

            local char = P.Character
            if char then
                local hum = char:FindFirstChildOfClass("Humanoid")
                if hum then hum.PlatformStand = false end
            end

            if counterConn then
                counterConn:Disconnect()
                counterConn = nil
            end

            for _, c in ipairs(autoResetConns) do
                pcall(function() c:Disconnect() end)
            end
            autoResetConns = {}

            for _, c in ipairs(autoLeaveConns) do
                pcall(function() c:Disconnect() end)
            end
            autoLeaveConns = {}
        end)
    end)
end

--enddefence tab

-- ═══════════════════════════════════════════════════════════════
-- ═══════════════════════════════════════════════════════════════
--target tab
-- TARGET TAB - FIXED WITH WORKING METHODS
-- ═══════════════════════════════════════════════════════════════
do
    local TargetTab = Tabs.Target
    local TPlayers = Players
    local TWorkspace = Workspace
    local TReplicatedStorage = ReplicatedStorage
    local TRunService = RunService
    local TDebris = Debris

    local TargetState = {
        SelectedPlayers = {},
        NonBlobMethod = "",
        BlobMethod = "",
        NonBlobActive = false,
        BlobActive = false,
        BlobKickHeight = 15,
        NonBlobOffsetX = 0,
        NonBlobOffsetY = -8,
        NonBlobOffsetZ = 0,
        AutoSitBlobmanActive = false,
        AutoSitBlobmanThread = nil,
    }

    local ActiveNonBlob = nil
    local ActiveBlob = nil

    local NonBlobTasks = {}
    local BlobTasks = {}

    local function GetPlayerList()
        local list = {}
        for _, p in ipairs(TPlayers:GetPlayers()) do
            if p ~= LocalPlayer then
                table.insert(list, p.DisplayName .. " (@" .. p.Name .. ")")
            end
        end
        table.sort(list)
        return list
    end

    function ExtractUsername(s)
        return s and s:match("@([%w_]+)")
    end

    function GetSelectedTargets()
        local out = {}
        for _, un in ipairs(TargetState.SelectedPlayers) do
            local plr = TPlayers:FindFirstChild(un)
            if plr and plr.Parent then
                table.insert(out, plr)
            end
        end
        return out
    end

    local LeftGroup = TargetTab:AddLeftGroupbox("Target Selection", "crosshair")
    local UtilityGroup = TargetTab:AddLeftGroupbox("Target Utilities", "wrench")

    local TargetCountLabel = LeftGroup:AddLabel("<b>Targets Selected:</b> <font color='#55e87a'>0</font>")

    local StopAllNonBlob, StartAllNonBlob
    local StopAllBlob, StartAllBlob
    local RagdollRemote = TReplicatedStorage:FindFirstChild("CharacterEvents")
        and TReplicatedStorage.CharacterEvents:FindFirstChild("RagdollRemote")

    -- ═══════════════════════════════════════════════════════════════
    -- SHARED PALLET RAGDOLL CORE (used by Loop Ragdoll toggle)
    -- Extracted from the Ownership Kick pallet mechanic.
    -- ═══════════════════════════════════════════════════════════════
    local PalletRagdoll = {
        Active = false,
        Pallet = nil,
        SoundPart = nil,
        SteppedConn = nil,
        CacheConn = nil,
        CleanupConn = nil,
        Thread = nil,
    }

    function PalletRagdoll.GetTarget()
        local targets = GetSelectedTargets()
        return targets[1]
    end

    function PalletRagdoll.FindPallet()
        local inv = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        return inv and inv:FindFirstChild("PalletLightBrown") or nil
    end

    function PalletRagdoll.SpawnPallet()
        if not PalletRagdoll.Active then return end
        if PalletRagdoll.Pallet and PalletRagdoll.Pallet.Parent then return end

        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        local SpawnToy = TReplicatedStorage:FindFirstChild("MenuToys")
            and TReplicatedStorage.MenuToys:FindFirstChild("SpawnToyRemoteFunction")
        if not SpawnToy then return end

        pcall(function()
            SpawnToy:InvokeServer(
                "PalletLightBrown",
                hrp.CFrame * CFrame.new(0, 10, 20),
                Vector3.zero
            )
        end)
    end

    function PalletRagdoll.ClearStepped()
        if PalletRagdoll.SteppedConn then
            pcall(function() PalletRagdoll.SteppedConn:Disconnect() end)
            PalletRagdoll.SteppedConn = nil
        end
    end

    function PalletRagdoll.Stop()
        PalletRagdoll.Active = false
        PalletRagdoll.ClearStepped()

        if PalletRagdoll.CacheConn then
            pcall(function() PalletRagdoll.CacheConn:Disconnect() end)
            PalletRagdoll.CacheConn = nil
        end
        if PalletRagdoll.CleanupConn then
            pcall(function() PalletRagdoll.CleanupConn:Disconnect() end)
            PalletRagdoll.CleanupConn = nil
        end

        local pallet = PalletRagdoll.Pallet
        if pallet and pallet.Parent then
            pcall(function()
                local destroy = TReplicatedStorage:FindFirstChild("MenuToys")
                    and TReplicatedStorage.MenuToys:FindFirstChild("DestroyToy")
                if destroy then destroy:FireServer(pallet) end
            end)
        end

        PalletRagdoll.Pallet = nil
        PalletRagdoll.SoundPart = nil

        -- Clean up leftovers
        local inv = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        if inv then
            for _, toy in ipairs(inv:GetChildren()) do
                if toy.Name == "PalletLightBrown" or toy.Name == "PalletForRagdoll" then
                    pcall(function()
                        local destroy = TReplicatedStorage:FindFirstChild("MenuToys")
                            and TReplicatedStorage.MenuToys:FindFirstChild("DestroyToy")
                        if destroy then destroy:FireServer(toy) end
                    end)
                end
            end
        end

        if PalletRagdoll.Thread then
            pcall(task.cancel, PalletRagdoll.Thread)
            PalletRagdoll.Thread = nil
        end
    end

    function PalletRagdoll.AttachToPallet(pallet)
        if not pallet or not pallet.Parent then return end

        local soundPart = pallet:WaitForChild("SoundPart", 3)
        if not soundPart then return end

        pcall(function()
            TReplicatedStorage.GrabEvents.SetNetworkOwner:FireServer(soundPart, soundPart.CFrame)
            TReplicatedStorage.GrabEvents.DestroyGrabLine:FireServer(soundPart)
        end)

        local partOwner = soundPart:WaitForChild("PartOwner", 1)
        if not (partOwner and partOwner.Value == LocalPlayer.Name) then
            pcall(function()
                TReplicatedStorage.MenuToys.DestroyToy:FireServer(pallet)
            end)
            return
        end

        for _, v in ipairs(pallet:GetChildren()) do
            if v:IsA("BasePart") then
                v.CanCollide = false
                v.CanQuery = false
                v.Transparency = 1
            end
        end

        pallet.Name = "PalletForRagdoll"
        PalletRagdoll.Pallet = pallet
        PalletRagdoll.SoundPart = soundPart

        local strikePhase = false

        PalletRagdoll.ClearStepped()
        PalletRagdoll.SteppedConn = TRunService.Stepped:Connect(function()
            if not PalletRagdoll.Active or not pallet.Parent or not soundPart.Parent then
                PalletRagdoll.ClearStepped()
                return
            end

            local target = PalletRagdoll.GetTarget()
            local tChar  = target and target.Character
            local tRoot  = tChar and tChar:FindFirstChild("HumanoidRootPart")
            local tHum   = tChar and tChar:FindFirstChildOfClass("Humanoid")

            if tRoot and tHum and tHum.Health > 0 then
                local ragdolledVal = tHum:FindFirstChild("Ragdolled")
                local isRagdolled = ragdolledVal and ragdolledVal.Value or false

                if not isRagdolled then
                    strikePhase = not strikePhase
                    if strikePhase then
                        soundPart.CFrame = tRoot.CFrame * CFrame.new(0, 2, 0)
                        soundPart.AssemblyLinearVelocity = Vector3.new(0, -9e5, 0)
                    else
                        soundPart.CFrame = tRoot.CFrame * CFrame.new(0, -1, 0)
                        soundPart.AssemblyLinearVelocity = Vector3.new(0, 9e5, 0)
                    end
                else
                    soundPart.CFrame = CFrame.new(0, 9e9, 0)
                    soundPart.AssemblyLinearVelocity = Vector3.zero
                end
            else
                soundPart.CFrame = CFrame.new(0, 9e9, 0)
                soundPart.AssemblyLinearVelocity = Vector3.zero
            end
        end)

        PalletRagdoll.CleanupConn = pallet.AncestryChanged:Connect(function()
            if pallet.Parent then return end
            PalletRagdoll.ClearStepped()
            PalletRagdoll.Pallet = nil
            PalletRagdoll.SoundPart = nil
            if PalletRagdoll.Active then
                task.wait(0.03)
                PalletRagdoll.SpawnPallet()
            end
        end)
    end

    function PalletRagdoll.Start()
        if PalletRagdoll.Active then return end
        if not PalletRagdoll.GetTarget() then
            Library:Notify({Description = "No target selected!", Time = 3})
            return
        end

        local inv = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        if not inv then
            Library:Notify({Description = "SpawnedInToys folder not found!", Time = 3})
            return
        end

        PalletRagdoll.Active = true
        PalletRagdoll.Pallet = nil
        PalletRagdoll.SoundPart = nil

        if PalletRagdoll.CacheConn then
            pcall(function() PalletRagdoll.CacheConn:Disconnect() end)
            PalletRagdoll.CacheConn = nil
        end

        PalletRagdoll.CacheConn = inv.ChildAdded:Connect(function(child)
            if not PalletRagdoll.Active then return end
            if child.Name ~= "PalletLightBrown" and child.Name ~= "PalletForRagdoll" then return end
            PalletRagdoll.AttachToPallet(child)
        end)

        -- Reuse existing pallet if one already exists
        local existing = PalletRagdoll.FindPallet()
        if existing then
            PalletRagdoll.AttachToPallet(existing)
        else
            PalletRagdoll.SpawnPallet()
        end
    end

    local TargetTabbox = TargetTab:AddRightTabbox("Target Methods")
    local NonBlobTab = TargetTabbox:AddTab("Non Blob", "user")
    local BlobmanTab = TargetTabbox:AddTab("Blobman", "bot")

    local TargetDropdown
    TargetDropdown = LeftGroup:AddDropdown("T_TargetSelect", {
        Text = "Target Player(s)",
        Values = GetPlayerList(),
        Default = nil,
        Multi = true,
        Searchable = true,
        Callback = function(selected)
            TargetState.SelectedPlayers = {}
            for entry, enabled in pairs(selected) do
                if enabled then
                    local un = ExtractUsername(entry)
                    if un then table.insert(TargetState.SelectedPlayers, un) end
                end
            end
            if TargetCountLabel then
                TargetCountLabel:SetText("<b>Targets Selected:</b> <font color='#55e87a'>" .. #TargetState.SelectedPlayers .. "</font>")
            end

            if TargetState.NonBlobActive and ActiveNonBlob then
                if StopAllNonBlob then StopAllNonBlob() end
                if StartAllNonBlob then StartAllNonBlob() end
            end
            if TargetState.BlobActive and ActiveBlob then
                if StopAllBlob then StopAllBlob() end
                if StartAllBlob then StartAllBlob() end
            end
        end,
    })

    LeftGroup:AddButton({
        Text = "Refresh List",
        Func = function()
            TargetDropdown:SetValues(GetPlayerList())
        end,
    })

    LeftGroup:AddButton({
        Text = "<b><font color='#ff5050'>Clear All Targets</font></b>",
        Func = function()
            TargetState.SelectedPlayers = {}
            TargetDropdown:SetValue({})
            if TargetCountLabel then
                TargetCountLabel:SetText("<b>Targets Selected:</b> <font color='#55e87a'>0</font>")
            end
            if TargetState.NonBlobActive and ActiveNonBlob and StopAllNonBlob then
                StopAllNonBlob()
            end
            if TargetState.BlobActive and ActiveBlob and StopAllBlob then
                StopAllBlob()
            end
        end,
    })

    LeftGroup:AddDivider()

    local DistLabel = LeftGroup:AddLabel("<b>Distance:</b> ---")

    task.spawn(function()
        while true do
            task.wait(0.5)
            local un = TargetState.SelectedPlayers[1]
            local target = un and TPlayers:FindFirstChild(un)
            if target and target.Character then
                local tHRP = target.Character:FindFirstChild("HumanoidRootPart")
                local mHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                if tHRP and mHRP then
                    local d = math.floor((tHRP.Position - mHRP.Position).Magnitude)
                    DistLabel:SetText("<b>Distance:</b> <font color='#55e87a'>" .. d .. " studs</font>")
                end
            else
                DistLabel:SetText("<b>Distance:</b> ---")
            end
        end
    end)

    -- ═══════════════════════════════════════════════════════════════
    -- METHOD IMPLEMENTATIONS
    -- ═══════════════════════════════════════════════════════════════

    local NonBlobMethods = {}
    local BlobMethods = {}

    -- ================================================================
    -- NON-BLOB METHODS
    -- ================================================================

    -- 1. LOOP GRAB KICK
    function Start_OPGrab(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            while state.active do
                task.wait(0.1)
                if not target or not target.Parent then break end
                local tChar = target.Character
                if not tChar then continue end
                local tRoot = tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar:FindFirstChild("Humanoid")
                if not (tRoot and tHum) or tHum.Health <= 0 then continue end
                local myChar = LocalPlayer.Character
                local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if not myRoot then continue end
                local offsetVec = Vector3.new(
                    TargetState.NonBlobOffsetX,
                    TargetState.NonBlobOffsetY,
                    TargetState.NonBlobOffsetZ
                )
                local saved = myRoot.CFrame
                myRoot.CFrame = CFrame.new(tRoot.Position + offsetVec)
                pcall(function() TReplicatedStorage.GrabEvents.SetNetworkOwner:FireServer(tRoot, tRoot.CFrame) end)
                task.wait(0.05)
                pcall(function() TReplicatedStorage.GrabEvents.DestroyGrabLine:FireServer(tRoot) end)
                task.wait(0.05)
                myRoot.CFrame = saved
            end
            state.active = false
        end)
    end
    function Stop_OPGrab(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

    -- 2. OATS KICK
    function Start_Oats(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local function snoLocal(part)
                pcall(function()
                    local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
                    local SNO = GE and GE:FindFirstChild("SetNetworkOwner")
                    if SNO and part then SNO:FireServer(part, part.CFrame) end
                end)
            end

            local myChar = LocalPlayer.Character
            local myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not (myChar and myHRP) then state.active = false; return end

            local savedPos = myHRP.CFrame
            local lastRemoteFire = tick()

            while state.active and TRunService.Heartbeat:Wait() do
                if not target or not target.Parent then break end

                myChar = LocalPlayer.Character
                myHRP = myChar and myChar:FindFirstChild("HumanoidRootPart")
                local myHead = myChar and myChar:FindFirstChild("Head")

                local tChar = target.Character
                local tHRP = tChar and tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar and tChar:FindFirstChild("Humanoid")

                if not (myChar and myHRP and myHead) or not (tHRP and tHum) or tHum.Health <= 0 then continue end

                local dist = (tHRP.Position - myHRP.Position).Magnitude

                if dist > 30 then
                    pcall(function() myChar:PivotTo(tHRP.CFrame * CFrame.new(0, 2, 4)) end)
                    snoLocal(tHRP)

                    if not tHRP:FindFirstChild("KickAlign") then
                        local oldBp = tHRP:FindFirstChildOfClass("BodyPosition")
                        if oldBp then oldBp:Destroy() end

                        local att0 = Instance.new("Attachment", tHRP)
                        att0.Name = "KickAtt0"
                        local att1 = Instance.new("Attachment", TWorkspace.Terrain)
                        att1.Name = "KickAtt1"

                        local alignPos = Instance.new("AlignPosition")
                        alignPos.Name = "KickAlign"
                        alignPos.Attachment0 = att0
                        alignPos.Attachment1 = att1
                        alignPos.MaxForce = math.huge
                        alignPos.Responsiveness = 200
                        alignPos.Parent = tHRP

                        local alignRot = Instance.new("AlignOrientation")
                        alignRot.Name = "KickRot"
                        alignRot.Attachment0 = att0
                        alignRot.Mode = Enum.OrientationAlignmentMode.OneAttachment
                        alignRot.MaxTorque = math.huge
                        alignRot.Responsiveness = 200
                        alignRot.Parent = tHRP
                    end

                    local grabStartTime = tick()
                    while (tick() - grabStartTime) < 0.3 and state.active do
                        task.wait(0.05)
                        snoLocal(tHRP)
                        pcall(function()
                            local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
                            local DL = GE and GE:FindFirstChild("DestroyGrabLine")
                            if DL then DL:FireServer(tHRP) end
                        end)
                        local align = tHRP:FindFirstChild("KickAlign")
                        if myHead and align and align.Attachment1 then
                            align.Attachment1.WorldPosition = myHead.Position + Vector3.new(0, 15, 0)
                        end
                    end

                    if state.active then
                        pcall(function()
                            myChar:PivotTo(savedPos)
                            tHRP.CFrame = savedPos * CFrame.new(0, 15, 0)
                        end)
                    end
                    continue
                end

                if not tHRP:FindFirstChild("KickAlign") then
                    local oldBp = tHRP:FindFirstChildOfClass("BodyPosition")
                    if oldBp then oldBp:Destroy() end

                    local att0 = Instance.new("Attachment", tHRP)
                    att0.Name = "KickAtt0"
                    local att1 = Instance.new("Attachment", TWorkspace.Terrain)
                    att1.Name = "KickAtt1"

                    local alignPos = Instance.new("AlignPosition")
                    alignPos.Name = "KickAlign"
                    alignPos.Attachment0 = att0
                    alignPos.Attachment1 = att1
                    alignPos.MaxForce = math.huge
                    alignPos.Responsiveness = 200
                    alignPos.Parent = tHRP

                    local alignRot = Instance.new("AlignOrientation")
                    alignRot.Name = "KickRot"
                    alignRot.Attachment0 = att0
                    alignRot.Mode = Enum.OrientationAlignmentMode.OneAttachment
                    alignRot.MaxTorque = math.huge
                    alignRot.Responsiveness = 200
                    alignRot.Parent = tHRP
                end

                snoLocal(tHRP)

                local align = tHRP:FindFirstChild("KickAlign")
                if align and align.Attachment1 and state.active then
                    align.Attachment1.WorldPosition = myHead.Position + Vector3.new(0, 20, 0)
                end

                local rot = tHRP:FindFirstChild("KickRot")
                if rot then rot.CFrame = CFrame.Angles(0, 0, 0) end

                if tick() - lastRemoteFire > 0.05 and state.active then
                    pcall(function()
                        local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
                        local DL = GE and GE:FindFirstChild("DestroyGrabLine")
                        if DL then DL:FireServer(tHRP) end
                    end)
                    lastRemoteFire = tick()
                end
            end

            if target and target.Parent and target.Character then
                local tH = target.Character:FindFirstChild("HumanoidRootPart")
                if tH then
                    for _, v in pairs(tH:GetChildren()) do
                        if v.Name == "KickAlign" or v.Name == "KickRot" or v.Name == "KickAtt0" then
                            pcall(function() v:Destroy() end)
                        end
                    end
                end
            end
            state.active = false
        end)
    end
    function Stop_Oats(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

     -- 3. OWNERSHIP KICK (Balanced Ownership Kick — Echo Hub port)
    function Start_Ownership(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local GE              = TReplicatedStorage:FindFirstChild("GrabEvents")
            if not GE then state.active = false; return end
            local SetNetOwner     = GE:FindFirstChild("SetNetworkOwner")
            local DestroyGrabLine = GE:FindFirstChild("DestroyGrabLine")
            if not SetNetOwner or not DestroyGrabLine then state.active = false; return end

            local ZERO_VECTOR = Vector3.new(0, 0, 0)

            local myChar = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
            local myRoot = myChar:WaitForChild("HumanoidRootPart", 5)
            if not myRoot then state.active = false; return end

            local savedPos = myRoot.CFrame

            local isGrabbing           = false
            local startTime            = nil
            local checkStartTime       = nil
            local grabAttemptStartTime = nil
            local currentTargetRoot    = nil
            local bodyPos              = nil
            local bodyGyro             = nil

            local function cleanupBodies()
                pcall(function()
                    if bodyPos  then bodyPos:Destroy()  bodyPos  = nil end
                    if bodyGyro then bodyGyro:Destroy() bodyGyro = nil end
                end)
            end

            local function cleanupAll()
                isGrabbing = false
                currentTargetRoot = nil
                checkStartTime = nil
                grabAttemptStartTime = nil
                cleanupBodies()
            end

            local function isTargetAlive()
                if not target or not target.Parent then return false end
                local char = target.Character
                if not char then return false end
                local hum = char:FindFirstChild("Humanoid")
                if not hum or hum.Health <= 0 then return false end
                local root = char:FindFirstChild("HumanoidRootPart")
                if not root then return false end
                if root ~= currentTargetRoot then currentTargetRoot = root end
                return true
            end

            local function createBodies(targetRoot, pos)
                cleanupBodies()

                for _, v in pairs(targetRoot:GetChildren()) do
                    if v:IsA("BodyPosition") or v:IsA("BodyGyro") then
                        pcall(function() v:Destroy() end)
                    end
                end

                bodyPos = Instance.new("BodyPosition")
                bodyPos.MaxForce = Vector3.new(9e9, 9e9, 9e9)
                bodyPos.D        = 500
                bodyPos.P        = 100000
                bodyPos.Position = pos
                bodyPos.Parent   = targetRoot

                bodyGyro = Instance.new("BodyGyro")
                bodyGyro.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
                bodyGyro.D         = 500
                bodyGyro.P         = 100000
                bodyGyro.CFrame    = CFrame.new(pos)
                bodyGyro.Parent    = targetRoot
            end

            while state.active and target and target.Parent do
                if not isTargetAlive() then
                    cleanupAll()
                    local waitStart = tick()
                    while state.active and target and target.Parent and tick() - waitStart < 0.5 do
                        if isTargetAlive() then break end
                        task.wait(0.1)
                    end
                    if not isTargetAlive() then
                        savedPos = myRoot.CFrame
                        startTime = nil
                        grabAttemptStartTime = nil
                        checkStartTime = nil
                    end
                    task.wait(0.1)
                    continue
                end

                myChar = LocalPlayer.Character
                myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if not myRoot then
                    task.wait()
                    continue
                end

                local tChar = target.Character
                local tRoot = currentTargetRoot
                local tHum  = tChar and tChar:FindFirstChild("Humanoid")

                if not (tRoot and tHum and tHum.Health > 0) then
                    task.wait()
                    continue
                end

                if not isGrabbing then
                    myRoot.CFrame = tRoot.CFrame * CFrame.new(0, 0, 3)
                    cleanupBodies()
                    checkStartTime = nil

                    pcall(function()
                        tHum.PlatformStand = true
                        tHum.Sit = true
                        SetNetOwner:FireServer(tRoot, tRoot.CFrame)
                        SetNetOwner:FireServer(tRoot, tRoot.CFrame)
                        DestroyGrabLine:FireServer(tRoot)
                    end)

                    myRoot.AssemblyLinearVelocity  = ZERO_VECTOR
                    myRoot.AssemblyAngularVelocity = ZERO_VECTOR

                    if grabAttemptStartTime == nil then grabAttemptStartTime = tick() end
                    if tick() - grabAttemptStartTime > 0.2 then
                        isGrabbing = true
                        grabAttemptStartTime = nil
                        checkStartTime = tick()
                        local lockPos = savedPos * CFrame.new(5, 20, 4)
                        createBodies(tRoot, lockPos.Position)
                    end

                else
                    myRoot.CFrame = savedPos
                    local lockPos = savedPos * CFrame.new(5, 20, 4)

                    myRoot.AssemblyLinearVelocity  = ZERO_VECTOR
                    myRoot.AssemblyAngularVelocity = ZERO_VECTOR

                    if bodyPos and bodyPos.Parent then
                        bodyPos.Position = lockPos.Position
                        if bodyGyro then
                            bodyGyro.CFrame = lockPos
                        end
                    else
                        createBodies(tRoot, lockPos.Position)
                    end

                    tHum.PlatformStand = true

                    pcall(function()
                        for i = 1, 4 do
                            SetNetOwner:FireServer(tRoot, lockPos)
                        end
                        DestroyGrabLine:FireServer(tRoot)
                    end)

                    if checkStartTime and tick() - checkStartTime > 0.2 then
                        local currentDist = (tRoot.Position - lockPos.Position).Magnitude

                        if currentDist > 15 then
                            cleanupAll()
                            myRoot.CFrame = tRoot.CFrame * CFrame.new(0, 0, 3)
                        else
                            checkStartTime = tick()
                        end
                    end
                end

                TRunService.Heartbeat:Wait()
            end

            cleanupAll()
            if myRoot and savedPos then
                myRoot.CFrame = savedPos
            end
            state.active = false
        end)
    end

    -- 3b. NEW OWNERSHIP KICK
    function Start_NewOwnership(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local NRS = TReplicatedStorage
            local NWS = TWorkspace
            local NLPlayers = TPlayers
            local NLPlayer  = LocalPlayer

            local GE = NRS:WaitForChild("GrabEvents")
            local SetNetOwner     = GE:WaitForChild("SetNetworkOwner")
            local DestroyGrabLine = GE:WaitForChild("DestroyGrabLine")

            local HIDDEN_CF   = CFrame.new(0, 1e9, 0)
            local ZERO_VECTOR = Vector3.new(0, 0, 0)

            local myChar = NLPlayer.Character
            local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not myRoot then state.active = false; return end

            local savedPos = myRoot.CFrame

            local isGrabbing           = false
            local startTime            = nil
            local lastPalletTime       = 0
            local checkStartTime       = nil
            local grabAttemptStartTime = nil
            local currentTargetRoot    = nil
            local attachments          = {}
            local spamConnection       = nil
            local spamActive           = true

            spamConnection = TRunService.Heartbeat:Connect(function()
                if not spamActive or not state.active then
                    if spamConnection then
                        pcall(function() spamConnection:Disconnect() end)
                        spamConnection = nil
                    end
                    return
                end
                if not target or not target.Parent then return end

                local tChar = target.Character
                local tRootActual = tChar and tChar:FindFirstChild("HumanoidRootPart")
                if tRootActual then
                    pcall(function()
                        SetNetOwner:FireServer(tRootActual, tRootActual.CFrame)
                        tRootActual.AssemblyLinearVelocity  = ZERO_VECTOR
                        tRootActual.AssemblyAngularVelocity = ZERO_VECTOR
                    end)
                end
            end)

            local function cleanupPhysicsObjects()
                isGrabbing = false
                currentTargetRoot = nil
                spamActive = false
                if spamConnection then
                    pcall(function() spamConnection:Disconnect() end)
                    spamConnection = nil
                end
                for _, v in pairs(attachments) do
                    if v then pcall(function() v:Destroy() end) end
                end
                attachments = {}
            end

            local function isTargetAlive()
                if not target or not target.Parent then return false end
                local char = target.Character
                if not char then return false end
                local hum = char:FindFirstChild("Humanoid")
                if not hum or hum.Health <= 0 then return false end
                local root = char:FindFirstChild("HumanoidRootPart")
                if not root then return false end
                if root ~= currentTargetRoot then currentTargetRoot = root end
                return true
            end

            local MyToys = NWS:FindFirstChild(NLPlayer.Name .. "SpawnedInToys")
            local pallet = MyToys and MyToys:FindFirstChild("PalletLightBrown")

            if not pallet then
                local canSpawn = NLPlayer:FindFirstChild("CanSpawnToy")
                if canSpawn and canSpawn.Value then
                    pcall(function()
                        NRS.MenuToys.SpawnToyRemoteFunction:InvokeServer(
                            "PalletLightBrown", HIDDEN_CF, Vector3.new(0, -90, 0)
                        )
                    end)
                    task.wait(0.5)
                    MyToys = NWS:FindFirstChild(NLPlayer.Name .. "SpawnedInToys")
                    pallet = MyToys and MyToys:FindFirstChild("PalletLightBrown")
                end
            end

            local soundPart = pallet and pallet:FindFirstChild("SoundPart")
            if not soundPart then
                warn("[Echo] New Ownership Kick: failed to get Pallet SoundPart")
                state.active = false
                cleanupPhysicsObjects()
                return
            end

            while state.active and target and target.Parent do
                TRunService.Heartbeat:Wait()

                if not isTargetAlive() then
                    cleanupPhysicsObjects()
                    local waitStart = tick()
                    while state.active and target and target.Parent
                        and tick() - waitStart < 0.5 do
                        if isTargetAlive() then break end
                        TRunService.Heartbeat:Wait()
                    end
                    if not isTargetAlive() then
                        savedPos = myRoot and myRoot.CFrame or savedPos
                        startTime = nil
                        grabAttemptStartTime = nil
                        checkStartTime = nil
                    end
                    continue
                end

                myChar = NLPlayer.Character
                myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if not myRoot then continue end

                local tChar = target.Character
                local tHum  = tChar and tChar:FindFirstChild("Humanoid")
                local tRoot = currentTargetRoot

                if tRoot and tHum then
                    pcall(function()
                        SetNetOwner:FireServer(tRoot, tRoot.CFrame)
                        DestroyGrabLine:FireServer(tRoot)
                    end)

                    local limbs = {"Left Leg", "Right Leg", "Left Arm", "Right Arm",
                                   "Head", "Torso", "UpperTorso", "LowerTorso"}
                    for _, limbName in ipairs(limbs) do
                        local part = tChar:FindFirstChild(limbName)
                        if part then
                            part.Velocity    = ZERO_VECTOR
                            part.RotVelocity = ZERO_VECTOR
                            part.CanCollide  = false
                        end
                    end
                end

                if not isGrabbing then
                    if tRoot and tHum then
                        myRoot.CFrame = tRoot.CFrame * CFrame.new(0, -8, -12)
                        myRoot.Velocity = ZERO_VECTOR

                        if not grabAttemptStartTime then grabAttemptStartTime = tick() end
                        tHum.PlatformStand = true
                        tHum.Sit = false

                        if not startTime then startTime = tick() end

                        if tick() - startTime > 0.05 then
                            isGrabbing = true
                            startTime = nil
                            grabAttemptStartTime = nil
                            myRoot.CFrame = savedPos
                            myRoot.Velocity = ZERO_VECTOR

                            local att0 = Instance.new("Attachment", tRoot)
                            local att1 = Instance.new("Attachment", myRoot)
                            att1.CFrame = CFrame.new(0, 14, 0)

                            local ap = Instance.new("AlignPosition", tRoot)
                            ap.Attachment0 = att0
                            ap.Attachment1 = att1
                            ap.MaxForce    = math.huge
                            ap.MaxVelocity = math.huge
                            ap.Responsiveness = 1000
                            ap.ApplyAtCenterOfMass = true

                            local ao = Instance.new("AlignOrientation", tRoot)
                            ao.Attachment0 = att0
                            ao.Attachment1 = att1
                            ao.MaxTorque   = math.huge
                            ao.Responsiveness = 1000

                            attachments.att0 = att0
                            attachments.att1 = att1
                            attachments.ap   = ap
                            attachments.ao   = ao

                            tHum:ChangeState(Enum.HumanoidStateType.Physics)
                        end
                    end
                else
                    if attachments.att1 then
                        attachments.att1.CFrame = CFrame.new(0, 18, 0)
                    end

                    if not isTargetAlive() then
                        cleanupPhysicsObjects()
                        myRoot.CFrame = savedPos
                        myRoot.Velocity = ZERO_VECTOR
                        continue
                    end

                    tRoot = currentTargetRoot
                    if not tRoot then continue end

                    if not checkStartTime then checkStartTime = tick() + 0.15 end
                    if checkStartTime and tick() >= checkStartTime then
                        local currentDistance = (myRoot.Position - tRoot.Position).Magnitude
                        if currentDistance > 20 then
                            cleanupPhysicsObjects()
                            isGrabbing = false
                            startTime = tick()
                            grabAttemptStartTime = nil
                            checkStartTime = nil
                            savedPos = myRoot.CFrame
                            myRoot.CFrame = tRoot.CFrame * CFrame.new(0, -10, 0)
                            myRoot.Velocity = ZERO_VECTOR
                            pcall(function()
                                SetNetOwner:FireServer(tRoot, tRoot.CFrame)
                                DestroyGrabLine:FireServer(tRoot)
                            end)
                            continue
                        end
                    end

                    pcall(function()
                        SetNetOwner:FireServer(tRoot, tRoot.CFrame)
                        DestroyGrabLine:FireServer(tRoot)
                    end)

                    local currentTime = tick()
                    if currentTime - lastPalletTime >= 0.05 and soundPart and soundPart.Parent then
                        lastPalletTime = currentTime
                        pcall(function()
                            soundPart.CFrame = tRoot.CFrame * CFrame.new(0, 2, 0)
                            SetNetOwner:FireServer(soundPart, soundPart.CFrame)
                            soundPart.CFrame = HIDDEN_CF
                        end)
                    end
                end
            end

            cleanupPhysicsObjects()
            if myRoot and myRoot.Parent and savedPos then
                myRoot.CFrame = savedPos
            end
            state.active = false
        end)
    end

    function Stop_Ownership(state)
        state.active = false
        if state.thread then
            pcall(task.cancel, state.thread)
            state.thread = nil
        end
    end

    function Stop_NewOwnership(state)
        state.active = false
        if state.thread then
            pcall(task.cancel, state.thread)
            state.thread = nil
        end
    end

    -- 5. LOOP KILL
    local HEIGHT_LIMIT = 100000
    local TELEPORT_OFFSET = Vector3.new(6, -18.5, 0)

    function isTooHigh(plr)
        local c = plr.Character
        local hrp = c and c:FindFirstChild("HumanoidRootPart")
        return not hrp or hrp.Position.Y > HEIGHT_LIMIT
    end

    function setNoCollideChar(char)
        for _, v in ipairs(char:GetDescendants()) do
            if v:IsA("BasePart") then v.CanCollide = false end
        end
    end

    function findBlobmanLocal()
        local toys = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
        return toys and toys:FindFirstChild("CreatureBlobman") or nil
    end

    function ensureBlobmanLocal()
        local b = findBlobmanLocal()
        if b then return b end
        pcall(function()
            local SpawnToy = TReplicatedStorage.MenuToys.SpawnToyRemoteFunction
            if SpawnToy and LocalPlayer.Character then
                SpawnToy:InvokeServer("CreatureBlobman", LocalPlayer.Character.HumanoidRootPart.CFrame * CFrame.new(0, 0, -5), Vector3.new(0, -15, 0))
            end
        end)
        for _ = 1, 30 do
            task.wait(0.1)
            b = findBlobmanLocal()
            if b then return b end
        end
        return nil
    end

    function modifyTarget(root, hum)
        if not (root and hum) or hum.Health <= 0 then return end
        local blob = ensureBlobmanLocal()
        if blob and blob:FindFirstChild("BlobmanSeatAndOwnerScript") then
            local drop = blob.BlobmanSeatAndOwnerScript:FindFirstChild("CreatureDrop")
            if drop then
                for _, part in ipairs(hum.Parent:GetDescendants()) do
                    if part:IsA("Weld") or part:IsA("BallSocketConstraint") then
                        drop:FireServer(part, part)
                    end
                end
            end
        end
        hum.Sit = false
        hum:ChangeState(Enum.HumanoidStateType.Running)
        hum:SetStateEnabled(Enum.HumanoidStateType.Seated, false)
        hum:ChangeState(Enum.HumanoidStateType.GettingUp)

        local plr = TPlayers:GetPlayerFromCharacter(hum.Parent)
        if plr and plr:FindFirstChild("IsHeld") then plr.IsHeld.Value = false end
        local rag = hum:FindFirstChild("Ragdolled")
        if rag then rag.Value = false end

        local bv = Instance.new("BodyVelocity")
        local bav = Instance.new("BodyAngularVelocity")
        bv.MaxForce = Vector3.new(1e7, -1e7, 1e7)
        bv.P = 100
        bv.Velocity = Vector3.new(math.random(-500, 50), -50, math.random(-50, 50))
        bav.MaxTorque = Vector3.new(-1e7, -1e7, -1e7)
        bav.P = 1e6
        bav.AngularVelocity = Vector3.new(math.random(-500, 300), math.random(-300, 300), math.random(-500, 500))
        bv.Parent = root
        bav.Parent = root
        hum.BreakJointsOnDeath = false
        hum:ChangeState(Enum.HumanoidStateType.Dead)
        task.delay(2, function()
            if bv.Parent then bv:Destroy() end
            if bav.Parent then bav:Destroy() end
        end)
    end

    function Start_LoopKill(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then state.active = false; return end

            local originalPos = hrp:GetPivot()

            while state.active do
                if not target or not target.Parent then break end
                local tChar = target.Character
                local tRoot = tChar and tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar and tChar:FindFirstChild("Humanoid")
                local tHead = tChar and tChar:FindFirstChild("Head")
                if not (tRoot and tHum and tHead) then task.wait(0.2); continue end
                if isTooHigh(target) then task.wait(0.2); continue end
                if tHum:GetState() == Enum.HumanoidStateType.Dead then task.wait(0.3); continue end

                char = LocalPlayer.Character
                hrp = char and char:FindFirstChild("HumanoidRootPart")
                if not hrp then task.wait(0.2); continue end

                hrp:PivotTo(CFrame.new(tRoot.Position + TELEPORT_OFFSET))
                setNoCollideChar(tChar)
                pcall(function() TReplicatedStorage.GrabEvents.SetNetworkOwner:FireServer(tRoot, tRoot.CFrame) end)
                task.wait(0.05)
                pcall(function() TReplicatedStorage.GrabEvents.DestroyGrabLine:FireServer(tRoot) end)
                task.wait(0.05)

                if tHead:FindFirstChild("PartOwner") and tHead.PartOwner.Value == LocalPlayer.Name then
                    task.wait(0.05)
                    modifyTarget(tRoot, tHum)
                end

                if hrp and originalPos then
                    hrp:PivotTo(originalPos)
                end

                task.wait(0.1)
            end
            state.active = false
        end)
    end
    function Stop_LoopKill(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
        local char = LocalPlayer.Character
        if char then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = true end
            end
        end
    end

    -- ================================================================
    -- BLOB METHODS
    -- ================================================================

    -- 1. QUICK KICK
    function Start_QuickKick(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local myChar = LocalPlayer.Character
            local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
            local tRoot = target.Character:FindFirstChild("HumanoidRootPart")
            local tHum = target.Character:FindFirstChild("Humanoid")
            if not (myRoot and tRoot and tHum) then state.active = false; return end
            local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
            local saved = myRoot.CFrame
            pcall(function()
                if GE and GE.SetNetworkOwner then GE.SetNetworkOwner:FireServer(tRoot, saved) end
                tHum.PlatformStand = true
                tRoot.CFrame = saved * CFrame.new(0, TargetState.BlobKickHeight, 0)
            end)
            task.wait(0.15)
            pcall(function()
                if GE and GE.DestroyGrabLine then GE.DestroyGrabLine:FireServer(tRoot) end
                tRoot.CFrame = saved * CFrame.new(0, -60, 0)
            end)
            task.wait(0.1)
            if myRoot then myRoot.CFrame = saved end
            state.active = false
        end)
    end
    function Stop_QuickKick(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

    -- 2. BLOBMAN KICK
    function Start_LoopKickBlob(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
            local myChar = LocalPlayer.Character
            local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not myRoot then state.active = false; return end

            local savedPos = myRoot.CFrame
            local dragging = false
            local grabStartTime = 0

            while state.active do
                if not target or not target.Parent or not target.Character then break end

                local tChar = target.Character
                local tRoot = tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar:FindFirstChild("Humanoid")
                local seat = myChar and myChar.Humanoid and myChar.Humanoid.SeatPart

                if tRoot and tHum and tHum.Health > 0 then
                    tRoot.AssemblyLinearVelocity = Vector3.zero
                    tRoot.Velocity = Vector3.zero

                    if seat then
                        local blobman = seat.Parent
                        local remoteFolder = blobman:FindFirstChild("BlobmanSeatAndOwnerScript")
                        local grab = remoteFolder and remoteFolder:FindFirstChild("CreatureGrab")
                        local drop = remoteFolder and remoteFolder:FindFirstChild("CreatureDrop")
                        local L_Det = blobman:FindFirstChild("LeftDetector")
                        local R_Det = blobman:FindFirstChild("RightDetector")
                        local L_Weld = L_Det and (L_Det:FindFirstChild("LeftWeld") or L_Det:FindFirstChild("RigidConstraint"))
                        local R_Weld = R_Det and (R_Det:FindFirstChild("RightWeld") or R_Det:FindFirstChild("RigidConstraint"))
                        if grab and drop and L_Weld and R_Weld then
                            pcall(function()
                                grab:FireServer(L_Det, tRoot, L_Weld)
                                grab:FireServer(R_Det, tRoot, R_Weld)
                                drop:FireServer(L_Weld, tRoot)
                                drop:FireServer(R_Weld, tRoot)
                            end)
                        end
                    end

                    if not dragging then
                        myRoot.CFrame = tRoot.CFrame
                        if GE then
                            pcall(function()
                                tHum.PlatformStand = true
                                if GE.SetNetworkOwner then GE.SetNetworkOwner:FireServer(tRoot, myRoot.CFrame) end
                                if GE.CreateGrabLine then GE.CreateGrabLine:FireServer(tRoot, Vector3.zero, tRoot.Position, false) end
                            end)
                        end
                        if grabStartTime == 0 then grabStartTime = tick() end
                        if tick() - grabStartTime > 0.3 then dragging = true; grabStartTime = 0 end
                    else
                        local lockPos = savedPos * CFrame.new(0, TargetState.BlobKickHeight, 0)
                        myRoot.CFrame = savedPos
                        tRoot.CFrame = lockPos
                        if GE then
                            pcall(function()
                                tHum.PlatformStand = true
                                if GE.SetNetworkOwner then GE.SetNetworkOwner:FireServer(tRoot, lockPos) end
                                if GE.DestroyGrabLine then GE.DestroyGrabLine:FireServer(tRoot) end
                                if GE.CreateGrabLine then GE.CreateGrabLine:FireServer(tRoot, Vector3.zero, tRoot.Position, false) end
                            end)
                        end
                    end
                else
                    dragging = false
                    grabStartTime = 0
                end
                TRunService.Heartbeat:Wait()
            end

            if myRoot and savedPos then myRoot.CFrame = savedPos end
            state.active = false
        end)
    end
    function Stop_LoopKickBlob(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

    -- 3. BLOBMAN LOCK
    function Start_BlobLock(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.thread = task.spawn(function()
            local CONFIG = {
                LockDistance = 9,
                LockHeight = 5,
                SlamLiftHeight = 20,
                SlamDropHeight = -3,
                GrabSpamSpeed = 0.02,
                LoopSpeed = 0.015,
                GrabDuration = 0.5,
                HitboxSize = 25,
                RagdollRepeat = 3,
                StruggleRepeat = 2,
                ResetWaitTime = 1.0,
                PCLDRetryInterval = 0.5,
            }

            local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
            local CharacterEvents = TReplicatedStorage:FindFirstChild("CharacterEvents")
            if not GE or not CharacterEvents then state.active = false; return end

            local RagdollRemote = CharacterEvents:FindFirstChild("RagdollRemote")
            local StruggleEvent = CharacterEvents:FindFirstChild("Struggle")
            local SetNetworkOwner = GE:FindFirstChild("SetNetworkOwner")
            local DestroyGrabLine = GE:FindFirstChild("DestroyGrabLine")
            local CreateGrabLine = GE:FindFirstChild("CreateGrabLine")

            local hitboxProxies = {}
            local hitboxOwnerChar = nil
            local PCLDCache = nil
            local LastSearch = 0
            local LastGrabTime = 0

            local function ClearHitboxProxies()
                for _, proxy in pairs(hitboxProxies) do
                    if proxy and proxy.Parent then
                        pcall(function() proxy:Destroy() end)
                    end
                end
                hitboxProxies = {}
                hitboxOwnerChar = nil
            end

            local function CreateHitboxForChar(tChar)
                if not tChar then return end
                if hitboxOwnerChar == tChar then return end
                ClearHitboxProxies()
                hitboxOwnerChar = tChar
                for _, obj in ipairs(tChar:GetDescendants()) do
                    if obj:IsA("BasePart") and obj.Name ~= "EchoHitboxProxy" then
                        local proxy = Instance.new("Part")
                        proxy.Name = "EchoHitboxProxy"
                        proxy.Size = Vector3.new(CONFIG.HitboxSize, CONFIG.HitboxSize, CONFIG.HitboxSize)
                        proxy.CFrame = obj.CFrame
                        proxy.Transparency = 1
                        proxy.CanCollide = false
                        proxy.CanTouch = true
                        proxy.CanQuery = true
                        proxy.Massless = true
                        proxy.Anchored = false
                        proxy.CastShadow = false
                        proxy.Parent = obj
                        local weld = Instance.new("WeldConstraint")
                        weld.Part0 = obj
                        weld.Part1 = proxy
                        weld.Parent = proxy
                        hitboxProxies[obj] = proxy
                    end
                end
            end

            local function GetPCLD(targetPlayer)
                if not targetPlayer then return nil end
                local char = targetPlayer.Character
                if not char then return nil end
                local knownNames = {
                    "PlayerCharacterLocationDetector", "PCld", "PCLD",
                    "LocationDetector", "CharacterLocationDetector",
                    "PlayerLocationDetector", "LocationPart", "Detector",
                    "TrackPart", "CharacterDetector",
                }
                for _, name in ipairs(knownNames) do
                    local found = char:FindFirstChild(name)
                    if found and found:IsA("BasePart") then return found end
                end
                for _, obj in ipairs(char:GetDescendants()) do
                    if obj:IsA("BasePart") then
                        local n = obj.Name:lower()
                        if n:find("location") or n:find("detect") or n:find("pcld") then
                            return obj
                        end
                    end
                end
                return nil
            end

            local function GetCurrentBlobman()
                local myChar = LocalPlayer.Character
                if not myChar then return nil end
                local hum = myChar:FindFirstChildOfClass("Humanoid")
                if not hum then return nil end
                local seatPart = hum.SeatPart
                if not seatPart then return nil end
                local parent = seatPart.Parent
                while parent do
                    if parent.Name == "CreatureBlobman" then
                        return parent
                    end
                    parent = parent.Parent
                end
                return nil
            end

            local function BlobGrabBothHands(blobman, tRoot)
                if not blobman or not tRoot then return end
                local remoteFolder = blobman:FindFirstChild("BlobmanSeatAndOwnerScript")
                if not remoteFolder then return end
                local grab = remoteFolder:FindFirstChild("CreatureGrab")
                local drop = remoteFolder:FindFirstChild("CreatureDrop")
                local L_Det = blobman:FindFirstChild("LeftDetector")
                local R_Det = blobman:FindFirstChild("RightDetector")
                local L_Weld = L_Det and (L_Det:FindFirstChild("LeftWeld") or L_Det:FindFirstChild("RigidConstraint"))
                local R_Weld = R_Det and (R_Det:FindFirstChild("RightWeld") or R_Det:FindFirstChild("RigidConstraint"))
                if grab and drop and L_Weld and R_Weld then
                    pcall(function()
                        grab:FireServer(L_Det, tRoot, L_Weld)
                        grab:FireServer(R_Det, tRoot, R_Weld)
                        drop:FireServer(L_Weld, tRoot)
                        drop:FireServer(R_Weld, tRoot)
                    end)
                end
            end

            local function LightRagdollSpam(tRoot)
                if not tRoot or not tRoot.Parent then return end
                for i = 1, CONFIG.RagdollRepeat do
                    pcall(function() RagdollRemote:FireServer(tRoot, 0) end)
                end
                for i = 1, CONFIG.StruggleRepeat do
                    pcall(function() StruggleEvent:FireServer(LocalPlayer) end)
                end
            end

            while state.active do
                if not target or not target.Parent then
                    task.wait(0.3)
                    continue
                end

                local tChar = target.Character
                if not tChar then task.wait(CONFIG.ResetWaitTime); continue end

                local tRoot = tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar:FindFirstChild("Humanoid")
                if not tRoot or not tHum then task.wait(0.3); continue end

                CreateHitboxForChar(tChar)

                if tHum.Health <= 0 then
                    LastGrabTime = 0
                    ClearHitboxProxies()
                    task.wait(CONFIG.ResetWaitTime)
                    continue
                end

                local blobman = GetCurrentBlobman()
                if not blobman or not blobman.Parent then
                    task.wait(0.5)
                    continue
                end

                local now = tick()
                if not PCLDCache or not PCLDCache.Parent or now - LastSearch > CONFIG.PCLDRetryInterval then
                    PCLDCache = GetPCLD(target)
                    LastSearch = now
                end

                local myChar = LocalPlayer.Character
                local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
                if not myRoot then task.wait(0.3); continue end

                local distance = (tRoot.Position - myRoot.Position).Magnitude
                if distance > 25 then
                    pcall(function()
                        local direction = (tRoot.Position - myRoot.Position).Unit
                        myRoot.CFrame = CFrame.new(tRoot.Position - direction * 15)
                        myRoot.AssemblyLinearVelocity = Vector3.zero
                        myRoot.AssemblyAngularVelocity = Vector3.zero
                    end)
                    task.wait(CONFIG.LoopSpeed)
                    continue
                end

                local myPos = myRoot.Position
                local lookDir = myRoot.CFrame.LookVector
                local targetPos = myPos + lookDir * CONFIG.LockDistance + Vector3.new(0, CONFIG.LockHeight, 0)
                local lookAtMe = CFrame.lookAt(targetPos, myPos)
                pcall(function()
                    tRoot.CFrame = lookAtMe
                    tRoot.AssemblyLinearVelocity = Vector3.zero
                    tRoot.AssemblyAngularVelocity = Vector3.zero
                end)

                BlobGrabBothHands(blobman, tRoot)

                pcall(function()
                    if SetNetworkOwner then SetNetworkOwner:FireServer(tRoot, tRoot.CFrame) end
                    if CreateGrabLine then CreateGrabLine:FireServer(tRoot, Vector3.zero, tRoot.Position, false) end
                    if DestroyGrabLine then DestroyGrabLine:FireServer(tRoot) end
                end)

                LightRagdollSpam(tRoot)

                if PCLDCache and PCLDCache.Parent then
                    pcall(function()
                        PCLDCache.CFrame = tRoot.CFrame
                        SetNetworkOwner:FireServer(PCLDCache, tRoot.CFrame)
                    end)
                end

                pcall(function() tHum.PlatformStand = true end)

                local now2 = tick()
                if now2 - LastGrabTime > CONFIG.GrabDuration then
                    local upPos = myPos + lookDir * CONFIG.LockDistance + Vector3.new(0, CONFIG.SlamLiftHeight, 0)
                    pcall(function()
                        tRoot.CFrame = CFrame.lookAt(upPos, myPos)
                        tRoot.AssemblyLinearVelocity = Vector3.zero
                        tRoot.AssemblyAngularVelocity = Vector3.zero
                    end)
                    task.wait(0.1)
                    local downPos = myPos + lookDir * CONFIG.LockDistance + Vector3.new(0, CONFIG.SlamDropHeight, 0)
                    pcall(function()
                        tRoot.CFrame = CFrame.lookAt(downPos, myPos)
                        tRoot.AssemblyLinearVelocity = Vector3.new(0, -50, 0)
                        tRoot.AssemblyAngularVelocity = Vector3.zero
                    end)
                    LastGrabTime = now2
                end

                task.wait(CONFIG.GrabSpamSpeed)
            end

            ClearHitboxProxies()
            state.active = false
        end)
    end
    function Stop_BlobLock(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

    -- 4. BLOBMAN KILL (OP) - core kill logic + method wrapper
    local BlobmanKillCore = {}
    BlobmanKillCore.Active = false
    BlobmanKillCore.Target = nil
    BlobmanKillCore.CurrentBlobman = nil
    BlobmanKillCore.Connection = nil
    BlobmanKillCore.KillCount = 0
    BlobmanKillCore.SelectedConnection = nil
    BlobmanKillCore.SelectedTask = nil
    BlobmanKillCore.AllTask = nil

    local BLOB_MAX_DISTANCE = 500
    local BLOB_GRAB_WAIT = 0.005
    local BLOB_RETRY_WAIT = 0.005
    local BLOB_MAX_RETRIES = 5

    function BlobmanKillCore.GetSeatedBlobman()
        local character = LocalPlayer.Character
        if not character then return nil end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return nil end
        local seat = humanoid.SeatPart
        if not seat or not seat:IsA("VehicleSeat") then return nil end
        local obj = seat
        while obj do
            if obj:IsA("Model") and obj.Name == "CreatureBlobman" then
                return obj
            end
            obj = obj.Parent
        end
        return nil
    end

    function BlobmanKillCore.SpawnBlobman()
        local seated = BlobmanKillCore.GetSeatedBlobman()
        if seated then
            BlobmanKillCore.CurrentBlobman = seated
            return seated
        end

        if BlobmanKillCore.CurrentBlobman and BlobmanKillCore.CurrentBlobman.Parent then
            local blobmanPos = nil
            if BlobmanKillCore.CurrentBlobman.PrimaryPart then
                blobmanPos = BlobmanKillCore.CurrentBlobman.PrimaryPart.Position
            else
                local part = BlobmanKillCore.CurrentBlobman:FindFirstChildWhichIsA("BasePart")
                if part then blobmanPos = part.Position end
            end

            local character = LocalPlayer.Character
            local rootPart = character and character:FindFirstChild("HumanoidRootPart")
            local localPos = rootPart and rootPart.Position

            if blobmanPos and localPos and (blobmanPos - localPos).Magnitude < BLOB_MAX_DISTANCE then
                local seat = BlobmanKillCore.CurrentBlobman:FindFirstChild("VehicleSeat")
                if seat then
                    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
                    if humanoid then
                        seat:Sit(humanoid)
                        task.wait(0.05)
                    end
                end
                return BlobmanKillCore.CurrentBlobman
            else
                pcall(function() BlobmanKillCore.CurrentBlobman:Destroy() end)
                BlobmanKillCore.CurrentBlobman = nil
            end
        end

        local character = LocalPlayer.Character
        if not character then return nil end
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if not rootPart then return nil end

        local spawnPos = rootPart.CFrame * CFrame.new(0, 0, -5)
        pcall(function()
            TReplicatedStorage.MenuToys.SpawnToyRemoteFunction:InvokeServer(
                "CreatureBlobman",
                spawnPos,
                Vector3.new(0, 127, 0)
            )
        end)

        local toyFolderName = LocalPlayer.Name .. "SpawnedInToys"
        local blobman = nil
        local startTime = tick()

        repeat
            local toyFolder = TWorkspace:FindFirstChild(toyFolderName)
            if toyFolder then
                blobman = toyFolder:FindFirstChild("CreatureBlobman")
            end
            if blobman then break end
            task.wait()
        until tick() - startTime > 2

        if not blobman then return nil end

        BlobmanKillCore.CurrentBlobman = blobman

        local seat = blobman:FindFirstChild("VehicleSeat")
        if seat then
            local humanoid = character:FindFirstChildOfClass("Humanoid")
            if humanoid then
                seat:Sit(humanoid)
            end
        end

        task.wait(0.05)
        return blobman
    end

    function BlobmanKillCore.KillPlayer(targetPlayer)
        if not targetPlayer or not targetPlayer.Character then return false end
        local humanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return false end

        for _ = 1, BLOB_MAX_RETRIES do
            local success = pcall(function()
                humanoid.BreakJointsOnDeath = false
                humanoid:ChangeState(Enum.HumanoidStateType.Dead)
            end)
            if success and humanoid.Health <= 0 then
                return true
            end
            task.wait(BLOB_RETRY_WAIT)
        end
        return false
    end

    function BlobmanKillCore.GrabRelease(blobman, targetRoot)
        if not blobman or not targetRoot then return end
        pcall(function()
            local script = blobman:FindFirstChild("BlobmanSeatAndOwnerScript")
            if script then
                script.CreatureGrab:FireServer(
                    blobman.LeftDetector,
                    targetRoot,
                    blobman.LeftDetector.LeftWeld
                )
                script.CreatureRelease:FireServer(blobman.LeftDetector.LeftWeld)
            end
        end)
    end

    function BlobmanKillCore.ProcessPlayer(targetPlayer)
        if not targetPlayer or not targetPlayer.Character then return false end

        local humanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
        if not humanoid or humanoid.Health <= 0 then return false end

        local targetRoot = targetPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not targetRoot then return false end

        local localChar = LocalPlayer.Character
        if not localChar then return false end

        local myRoot = localChar:FindFirstChild("HumanoidRootPart")
        if not myRoot then return false end

        local originalCFrame = myRoot.CFrame
        local originalVel = myRoot.AssemblyLinearVelocity
        local originalAngVel = myRoot.AssemblyAngularVelocity

        pcall(function()
            myRoot.CFrame = targetRoot.CFrame
            myRoot.AssemblyLinearVelocity = Vector3.zero
            myRoot.AssemblyAngularVelocity = Vector3.zero

            TRunService.Heartbeat:Wait()

            humanoid.BreakJointsOnDeath = false
            humanoid:ChangeState(Enum.HumanoidStateType.Dead)

            local blobman = BlobmanKillCore.GetSeatedBlobman()
            if blobman then
                BlobmanKillCore.GrabRelease(blobman, targetRoot)
            end
        end)

        if myRoot and myRoot.Parent then
            myRoot.CFrame = originalCFrame
            myRoot.AssemblyLinearVelocity = originalVel or Vector3.zero
            myRoot.AssemblyAngularVelocity = originalAngVel or Vector3.zero
            myRoot.Velocity = Vector3.zero
            myRoot.RotVelocity = Vector3.zero
        end

        return true
    end

    function BlobmanKillCore.Stop()
        BlobmanKillCore.Active = false
        BlobmanKillCore.Target = nil
        BlobmanKillCore.Connection = nil
        if BlobmanKillCore.CurrentBlobman and not BlobmanKillCore.CurrentBlobman.Parent then
            BlobmanKillCore.CurrentBlobman = nil
        end
    
        if BlobKillOP then BlobKillOP.Running = false end
    end

    function BlobmanKillCore.ProcessAllPlayers()
        local localChar = LocalPlayer.Character
        if not localChar or not localChar:FindFirstChild("HumanoidRootPart") then return end

        local rootPart = localChar.HumanoidRootPart
        local blobman = BlobmanKillCore.SpawnBlobman()
        if not blobman then return end

        local targets = {}
        for _, player in ipairs(TPlayers:GetPlayers()) do
            if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                local hum = player.Character:FindFirstChildOfClass("Humanoid")
                if hum and hum.Health > 0 then
                    table.insert(targets, player)
                end
            end
        end

        for _, player in ipairs(targets) do
            if not BlobmanKillCore.Active then break end

            local targetChar = player.Character
            if not targetChar then continue end

            local humanoid = targetChar:FindFirstChildOfClass("Humanoid")
            if not humanoid or humanoid.Health <= 0 then continue end

            local targetRoot = targetChar:FindFirstChild("HumanoidRootPart")
            if not targetRoot then continue end

            rootPart.CFrame = targetRoot.CFrame
            task.wait(0.02)

            BlobmanKillCore.KillPlayer(player)
            BlobmanKillCore.KillCount = BlobmanKillCore.KillCount + 1

            for _ = 1, 2 do
                BlobmanKillCore.GrabRelease(blobman, targetRoot)
                task.wait(BLOB_GRAB_WAIT)
            end
        end
    end

    -- The apply-method state controller for Blobman Kill (OP)
    -- This is shared between the Kill Selected button, Kill All button,
    -- and the Apply Blobman Method toggle so all three stay in sync.
    local BlobKillOP = {
        Running = false,
        Mode = nil, -- "selected" or "all"
    }

    function BlobKillOP.StopAll()
        BlobKillOP.Running = false
        BlobmanKillCore.Stop()

        if BlobmanKillCore.SelectedConnection then
            pcall(function() BlobmanKillCore.SelectedConnection:Disconnect() end)
            BlobmanKillCore.SelectedConnection = nil
        end
        if BlobmanKillCore.SelectedTask then
            pcall(task.cancel, BlobmanKillCore.SelectedTask)
            BlobmanKillCore.SelectedTask = nil
        end
        if BlobmanKillCore.AllTask then
            pcall(task.cancel, BlobmanKillCore.AllTask)
            BlobmanKillCore.AllTask = nil
        end

    end

    -- Loop processing of selected targets (used by Kill Selected button and Apply toggle)
    function BlobKillOP.StartSelectedLoop()
        BlobKillOP.StopAll()
        BlobKillOP.Running = true
        BlobKillOP.Mode = "selected"
        BlobmanKillCore.Active = true
        BlobmanKillCore.SpawnBlobman()

        BlobmanKillCore.SelectedTask = task.spawn(function()
            while BlobKillOP.Running and BlobmanKillCore.Active do
                local targets = GetSelectedTargets()
                if #targets == 0 then
                    -- If apply method is on and the user cleared targets, keep looping quietly
                    task.wait(0.25)
                else
                    for _, target in ipairs(targets) do
                        if not BlobKillOP.Running or not BlobmanKillCore.Active then break end
                        if target and target.Parent then
                            pcall(BlobmanKillCore.ProcessPlayer, target)
                        end
                    end
                    task.wait()
                end
            end
        end)

        -- Reattach on character respawn for each selected target
        BlobmanKillCore.SelectedConnection = TPlayers.PlayerAdded:Connect(function(plr)
            local conn
            conn = plr.CharacterAdded:Connect(function()
                if not BlobKillOP.Running then
                    if conn then conn:Disconnect() end
                    return
                end
                task.spawn(function()
                    if table.find(TargetState.SelectedPlayers, plr.Name) then
                        pcall(BlobmanKillCore.ProcessPlayer, plr)
                    end
                end)
            end)
        end)
    end

    -- Method wrapper used by Apply Blobman Method toggle
    function Start_NewBlobKill(target, state)
        if not target or not target.Character then return end
        state.active = true
        if not BlobKillOP.Running then
            BlobKillOP.StartSelectedLoop()
        end
    end

    function Stop_NewBlobKill(state)
        state.active = false
        BlobKillOP.StopAll()
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
    end

    -- 5. SPIN LOOP KICK
    function Start_Spin(target, state)
        if not target or not target.Character then return end
        state.active = true
        state.angle = 0
        state.radius = 25
        state.speed = 0.25
        state.height = 20
        state.thread = task.spawn(function()
            local GE = TReplicatedStorage:FindFirstChild("GrabEvents")
            local myChar = LocalPlayer.Character
            local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not myRoot then state.active = false; return end

            local savedPos = myRoot.CFrame
            local dragging = false
            local grabStartTime = 0

            while state.active do
                if not target or not target.Parent or not target.Character then break end

                local tChar = target.Character
                local tRoot = tChar:FindFirstChild("HumanoidRootPart")
                local tHum = tChar:FindFirstChild("Humanoid")
                local seat = myChar and myChar.Humanoid and myChar.Humanoid.SeatPart

                if tRoot and tHum and tHum.Health > 0 then
                    tRoot.AssemblyLinearVelocity = Vector3.zero
                    tRoot.Velocity = Vector3.zero

                    if seat then
                        local blobman = seat.Parent
                        local remoteFolder = blobman:FindFirstChild("BlobmanSeatAndOwnerScript")
                        local grab = remoteFolder and remoteFolder:FindFirstChild("CreatureGrab")
                        local drop = remoteFolder and remoteFolder:FindFirstChild("CreatureDrop")
                        local L_Det = blobman:FindFirstChild("LeftDetector")
                        local R_Det = blobman:FindFirstChild("RightDetector")
                        local L_Weld = L_Det and (L_Det:FindFirstChild("LeftWeld") or L_Det:FindFirstChild("RigidConstraint"))
                        local R_Weld = R_Det and (R_Det:FindFirstChild("RightWeld") or R_Det:FindFirstChild("RigidConstraint"))
                        if grab and drop and L_Weld and R_Weld then
                            pcall(function()
                                grab:FireServer(L_Det, tRoot, L_Weld)
                                grab:FireServer(R_Det, tRoot, R_Weld)
                                drop:FireServer(L_Weld, tRoot)
                                drop:FireServer(R_Weld, tRoot)
                            end)
                        end
                    end

                    state.angle = state.angle + state.speed
                    if state.angle > 6.28 then state.angle = 0 end
                    local x = math.cos(state.angle) * state.radius
                    local z = math.sin(state.angle) * state.radius

                    if not dragging then
                        myRoot.CFrame = tRoot.CFrame * CFrame.new(x, 0, z)
                        if GE then
                            pcall(function()
                                tHum.PlatformStand = true
                                if GE.SetNetworkOwner then GE.SetNetworkOwner:FireServer(tRoot, myRoot.CFrame) end
                                if GE.CreateGrabLine then GE.CreateGrabLine:FireServer(tRoot, Vector3.zero, tRoot.Position, false) end
                            end)
                        end
                        if grabStartTime == 0 then grabStartTime = tick() end
                        if tick() - grabStartTime > 0.3 then dragging = true; grabStartTime = 0 end
                    else
                        myRoot.CFrame = tRoot.CFrame * CFrame.new(x, 0, z)
                        local lockPos = savedPos * CFrame.new(0, state.height, 0)
                        tRoot.CFrame = lockPos
                        if GE then
                            pcall(function()
                                tHum.PlatformStand = true
                                if GE.SetNetworkOwner then GE.SetNetworkOwner:FireServer(tRoot, lockPos) end
                                if GE.DestroyGrabLine then GE.DestroyGrabLine:FireServer(tRoot) end
                                if GE.CreateGrabLine then GE.CreateGrabLine:FireServer(tRoot, Vector3.zero, tRoot.Position, false) end
                            end)
                        end
                    end
                else
                    dragging = false
                    grabStartTime = 0
                end
                TRunService.Heartbeat:Wait()
            end

            if myRoot and savedPos then myRoot.CFrame = savedPos end
            state.active = false
        end)
    end
    function Stop_Spin(state)
        state.active = false
        if state.thread then pcall(task.cancel, state.thread); state.thread = nil end
        state.angle = 0
    end

print("[Echo debug] Start_Ownership is", Start_Ownership and "present" or "NIL")
print("[Echo debug] Stop_Ownership is", Stop_Ownership and "present" or "NIL")
print("[Echo debug] Start_NewOwnership is", Start_NewOwnership and "present" or "NIL")
print("[Echo debug] Stop_NewOwnership is", Stop_NewOwnership and "present" or "NIL")

    -- Register methods
    NonBlobMethods["Oats Kick"]          = { Start = Start_Oats,          Stop = Stop_Oats }
    NonBlobMethods["Ownership Kick"]     = { Start = Start_Ownership,     Stop = Stop_Ownership }
    NonBlobMethods["Ownership Kick v2"]  = { Start = Start_NewOwnership,  Stop = Stop_NewOwnership }
    NonBlobMethods["Loop Kill"]          = { Start = Start_LoopKill,      Stop = Stop_LoopKill }

    BlobMethods["Quick Kick"]            = { Start = Start_QuickKick,    Stop = Stop_QuickKick }
    BlobMethods["Blobman Kick"]          = { Start = Start_LoopKickBlob, Stop = Stop_LoopKickBlob }
    BlobMethods["Blobman Lock"]          = { Start = Start_BlobLock,     Stop = Stop_BlobLock }
    BlobMethods["Blobman Kill (OP)"]     = { Start = Start_NewBlobKill,  Stop = Stop_NewBlobKill }
    BlobMethods["Spin Loop Kick"]        = { Start = Start_Spin,         Stop = Stop_Spin }

    StopAllNonBlob = function()
        for un, entry in pairs(NonBlobTasks) do
            if entry.stop and entry.state then
                pcall(entry.stop, entry.state)
            end
            NonBlobTasks[un] = nil
        end
    end

    StartAllNonBlob = function()
        StopAllNonBlob()
        if not ActiveNonBlob then return end
        local method = NonBlobMethods[ActiveNonBlob]
        if not method then return end
        local targets = GetSelectedTargets()
        if #targets == 0 then
            Library:Notify({Description = "No valid targets selected!", Time = 3})
            return
        end
        for _, target in ipairs(targets) do
            local state = { active = false, thread = nil }
            NonBlobTasks[target.Name] = { state = state, stop = method.Stop }
            method.Start(target, state)
        end
    end

    StopAllBlob = function()
        for un, entry in pairs(BlobTasks) do
            if entry.stop and entry.state then
                pcall(entry.stop, entry.state)
            end
            BlobTasks[un] = nil
        end
        -- Blobman Kill (OP) uses a shared loop; make sure it fully clears
        BlobKillOP.StopAll()
    end

    StartAllBlob = function()
        StopAllBlob()
        if not ActiveBlob then return end
        local method = BlobMethods[ActiveBlob]
        if not method then return end

        -- Blobman Kill (OP): run the shared selected loop once (not once-per-target),
        -- so both the toggle and the Kill Selected button operate on the same loop.
        if ActiveBlob == "Blobman Kill (OP)" then
            BlobKillOP.StartSelectedLoop()
            return
        end

        local targets = GetSelectedTargets()
        if #targets == 0 then
            Library:Notify({Description = "No valid targets selected!", Time = 3})
            return
        end
        for _, target in ipairs(targets) do
            local state = { active = false, thread = nil }
            BlobTasks[target.Name] = { state = state, stop = method.Stop }
            method.Start(target, state)
        end
    end

    -- ================================================================
    -- NON-BLOB DROPDOWN + APPLY
    -- ================================================================
    NonBlobTab:AddDropdown("T_NonBlobMethod", {
        Text = "Non-Blobman Method",
        Values = {
            "Oats Kick",
            "Ownership Kick",
            "Ownership Kick v2",
            "Loop Kill",
        },
        Default = "",
        AllowNull = true,
        Callback = function(v)
            TargetState.NonBlobMethod = v
            if TargetState.NonBlobActive then
                StopAllNonBlob()
                ActiveNonBlob = nil
                if v ~= "" and NonBlobMethods[v] then
                    ActiveNonBlob = v
                    StartAllNonBlob()
                end
            end
        end,
    })

    NonBlobTab:AddToggle("T_NonBlobApply", {
        Text = "<b>Apply Non-Blobman Method</b>",
        Default = false,
        Callback = function(v)
            TargetState.NonBlobActive = v
            if v then
                if TargetState.NonBlobMethod == "" or not NonBlobMethods[TargetState.NonBlobMethod] then
                    Library:Notify({Description = "Select a Non-Blobman method first!", Time = 3})
                    Toggles.T_NonBlobApply:SetValue(false)
                    TargetState.NonBlobActive = false
                    return
                end
                ActiveNonBlob = TargetState.NonBlobMethod
                StartAllNonBlob()
            else
                StopAllNonBlob()
                ActiveNonBlob = nil
            end
        end,
    })

    NonBlobTab:AddDivider()

    NonBlobTab:AddSlider("T_NonBlobOffsetX", {
        Text = "Non-Blob Offset X",
        Default = 0,
        Min = -20,
        Max = 20,
        Rounding = 1,
        Callback = function(v) TargetState.NonBlobOffsetX = v end,
    })

    NonBlobTab:AddSlider("T_NonBlobOffsetY", {
        Text = "Non-Blob Offset Y",
        Default = -8,
        Min = -15,
        Max = -2,
        Rounding = 1,
        Callback = function(v) TargetState.NonBlobOffsetY = v end,
    })

    NonBlobTab:AddSlider("T_NonBlobOffsetZ", {
        Text = "Non-Blob Offset Z",
        Default = 0,
        Min = -20,
        Max = 20,
        Rounding = 1,
        Callback = function(v) TargetState.NonBlobOffsetZ = v end,
    })

    -- ================================================================
    -- BLOB DROPDOWN + APPLY
    -- ================================================================
    BlobmanTab:AddDropdown("T_BlobMethod", {
        Text = "Blobman Method",
        Values = {
            "Quick Kick",
            "Blobman Kick",
            "Blobman Lock",
            "Blobman Kill (OP)",
            "Spin Loop Kick",
        },
        Default = "",
        AllowNull = true,
        Callback = function(v)
            TargetState.BlobMethod = v
            if TargetState.BlobActive then
                StopAllBlob()
                ActiveBlob = nil
                if v ~= "" and BlobMethods[v] then
                    ActiveBlob = v
                    StartAllBlob()
                end
            end
        end,
    })

    BlobmanTab:AddToggle("T_BlobApply", {
        Text = "<b>Apply Blobman Method</b>",
        Default = false,
        Callback = function(v)
            TargetState.BlobActive = v
            if v then
                if TargetState.BlobMethod == "" or not BlobMethods[TargetState.BlobMethod] then
                    Library:Notify({Description = "Select a Blobman method first!", Time = 3})
                    Toggles.T_BlobApply:SetValue(false)
                    TargetState.BlobActive = false
                    return
                end
                ActiveBlob = TargetState.BlobMethod
                StartAllBlob()
            else
                StopAllBlob()
                ActiveBlob = nil
            end
        end,
    })

    BlobmanTab:AddToggle("T_AutoSitBlobman", {
        Text = "Auto Sit on Blobman",
        Default = false,
        Callback = function(v)
            TargetState.AutoSitBlobmanActive = v and true or false
            if TargetState.AutoSitBlobmanThread then
                pcall(task.cancel, TargetState.AutoSitBlobmanThread)
                TargetState.AutoSitBlobmanThread = nil
            end
            if not TargetState.AutoSitBlobmanActive then return end

            TargetState.AutoSitBlobmanThread = task.spawn(function()
                local spawnRemote = TReplicatedStorage:FindFirstChild("MenuToys") and TReplicatedStorage.MenuToys:FindFirstChild("SpawnToyRemoteFunction")
                while TargetState.AutoSitBlobmanActive do
                    pcall(function()
                        local char = LocalPlayer.Character
                        local hum = char and char:FindFirstChildOfClass("Humanoid")
                        local hrp = char and char:FindFirstChild("HumanoidRootPart")
                        if not hum or not hrp or hum.SeatPart then return end

                        local folder = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                        local blob = folder and folder:FindFirstChild("CreatureBlobman")
                        if not blob and spawnRemote then
                            spawnRemote:InvokeServer("CreatureBlobman", hrp.CFrame * CFrame.new(0, 5, 5), Vector3.zero)
                            local started = tick()
                            repeat
                                TRunService.Heartbeat:Wait()
                                folder = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                                blob = folder and folder:FindFirstChild("CreatureBlobman")
                            until blob or tick() - started > 5 or not TargetState.AutoSitBlobmanActive
                        end

                        if blob and TargetState.AutoSitBlobmanActive then
                            local seat = blob:FindFirstChildWhichIsA("VehicleSeat", true) or blob:FindFirstChildWhichIsA("Seat", true)
                            if seat then
                                hrp.CFrame = seat.CFrame * CFrame.new(0, 1, 0)
                                hrp.AssemblyLinearVelocity = Vector3.zero
                                hrp.AssemblyAngularVelocity = Vector3.zero
                                pcall(function() seat:Sit(hum) end)
                            end
                        end
                    end)
                    task.wait(0.1)
                end
                TargetState.AutoSitBlobmanThread = nil
            end)
        end,
    })

    local BlobmanIcon = nil
    pcall(function()
        local url = "https://files.catbox.moe/f5wy7q.png"
        local fileName = "Echo_BlobmanIcon_f5wy7q.png"
        local okWrite = pcall(function()
            local http = _G.__EchoHttpGet or function(u) return game:HttpGet(u) end
            if type(writefile) == "function" and (type(isfile) ~= "function" or not isfile(fileName)) then
                writefile(fileName, http(url))
            end
        end)
        local getter = rawget(_G, "getcustomasset") or rawget(_G, "getsynasset") or rawget(_G, "getasset")
        if okWrite and type(getter) == "function" and type(isfile) == "function" and isfile(fileName) then
            local okAsset, asset = pcall(getter, fileName)
            if okAsset and type(asset) == "string" and asset ~= "" then
                BlobmanIcon = asset
            end
        end
    end)

    BlobmanTab:AddButton({
        Text = "Spawn & Sit Blobman",
        Image = BlobmanIcon,
        Func = function()
            local char = LocalPlayer.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp or not hum then return end

            local folder = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
            local blob = folder and folder:FindFirstChild("CreatureBlobman")

            if blob then
                local seat = blob:FindFirstChildWhichIsA("VehicleSeat", true)
                if seat then
                    hrp.CFrame = seat.CFrame * CFrame.new(0, 1, 0)
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.AssemblyAngularVelocity = Vector3.zero
                    pcall(function() seat:Sit(hum) end)
                end
                return
            end

            local spawnRemote = TReplicatedStorage:FindFirstChild("MenuToys") and TReplicatedStorage.MenuToys:FindFirstChild("SpawnToyRemoteFunction")
            if not spawnRemote then return end

            local connection
            if folder then
                connection = folder.ChildAdded:Connect(function(child)
                    if child.Name ~= "CreatureBlobman" then return end
                    if connection then connection:Disconnect(); connection = nil end
                    task.spawn(function()
                        local seat = child:FindFirstChildWhichIsA("VehicleSeat", true) or child:FindFirstChildWhichIsA("Seat", true)
                        if seat and hum and hum.Parent then
                            hrp.CFrame = seat.CFrame * CFrame.new(0, 1, 0)
                            task.wait(0.1)
                            pcall(function() seat:Sit(hum) end)
                        end
                    end)
                end)
            end

            pcall(function()
                spawnRemote:InvokeServer("CreatureBlobman", hrp.CFrame, Vector3.zero)
            end)

            task.delay(5, function()
                if connection then
                    connection:Disconnect()
                    connection = nil
                end
            end)
        end,
    })

    -- Kill Selected uses the same Apply Blobman Method state.
    BlobmanTab:AddButton({
        Text = "Kill Selected",
        Func = function()
            local targets = GetSelectedTargets()
            if #targets == 0 then
                Library:Notify({Description = "No valid targets selected!", Time = 3})
                return
            end

            TargetState.BlobMethod = "Blobman Kill (OP)"
            ActiveBlob = "Blobman Kill (OP)"
            pcall(function()
                Toggles.T_BlobApply:SetValue(not Toggles.T_BlobApply.Value)
            end)
        end,
    })

    BlobmanTab:AddDivider()

    BlobmanTab:AddSlider("T_BlobKickHeight", {
        Text = "Blobman Kick Height",
        Default = 15,
        Min = 10,
        Max = 25,
        Rounding = 0,
        Suffix = " studs",
        Callback = function(v)
            TargetState.BlobKickHeight = v
        end,
    })

    -- Utility toggles
    local removeGucciActive = false
    local destroyFoodActive = false
    local removeAntiKickStickyActive = false

    function getPrimaryTarget()
        local targets = GetSelectedTargets()
        return targets[1]
    end

    UtilityGroup:AddToggle("T_LoopRagdoll", {
        Text = "Loop Ragdoll",
        Default = false,
        Tooltip = "Pallet-based ragdoll loop (same mechanic as Ownership Kick)",
        Callback = function(v)
            if v then
                PalletRagdoll.Start()
                -- If the core refused to start (no target), reflect that in the UI.
                if not PalletRagdoll.Active then
                    Toggles.T_LoopRagdoll:SetValue(false)
                end
            else
                PalletRagdoll.Stop()
            end
        end,
    })

      UtilityGroup:AddToggle("T_RemoveGucci", {
        Text = "Remove/Destroy Gucci",
        Default = false,
        Callback = function(v)
            removeGucciActive = v and true or false
            if not removeGucciActive then return end

            local ATTEMPT_TIME = 0.9
            local COOLDOWN     = 0.45

            task.spawn(function()
                while removeGucciActive do
                    local target = getPrimaryTarget()
                    if not target or not target.Parent then
                        -- No target selected — keep looping quietly instead
                        -- of killing the toggle. The user may pick one later.
                        task.wait(0.5)
                        continue
                    end

                    local myCharacter = LocalPlayer.Character
                    local myHumanoid  = myCharacter and myCharacter:FindFirstChildOfClass("Humanoid")
                    local myRoot      = myCharacter and myCharacter:FindFirstChild("HumanoidRootPart")
                    local targetCharacter = target.Character
                    local targetHumanoid  = targetCharacter and targetCharacter:FindFirstChildOfClass("Humanoid")

                    if not myHumanoid or not myRoot or not targetHumanoid then
                        task.wait(0.5)
                        continue
                    end

                    local targetSeat = targetHumanoid.SeatPart

                    if targetSeat and targetSeat:IsA("Seat") then
                        local returnCFrame = myRoot.CFrame

                        if myHumanoid.SeatPart ~= targetSeat then
                            local magnetConnection = RunService.Stepped:Connect(function()
                                if not removeGucciActive or not targetSeat.Parent or not myRoot.Parent then
                                    return
                                end
                                myRoot.CFrame = targetSeat.CFrame
                                myRoot.AssemblyLinearVelocity = Vector3.zero
                                if targetSeat.Parent and targetSeat.Parent.PrimaryPart then
                                    targetSeat.Parent.PrimaryPart.AssemblyLinearVelocity = Vector3.zero
                                    targetSeat.Parent.PrimaryPart.AssemblyAngularVelocity = Vector3.zero
                                end
                            end)

                            local deadline = os.clock() + ATTEMPT_TIME

                            while removeGucciActive and os.clock() < deadline and targetSeat.Parent and myHumanoid.SeatPart ~= targetSeat do
                                pcall(function()
                                    targetSeat:Sit(myHumanoid)
                                end)
                                task.wait()
                            end

                            magnetConnection:Disconnect()

                            if myHumanoid.SeatPart == targetSeat then
                                task.wait(0.15)
                                myHumanoid.Sit = false
                                myHumanoid.Jump = true
                                task.wait(0.05)

                                if myRoot.Parent then
                                    myRoot.CFrame = returnCFrame
                                    myRoot.AssemblyLinearVelocity = Vector3.zero
                                end

                                Library:Notify({
                                    Title = "Echo",
                                    Description = target.DisplayName .. "'s vehicle has been removed!",
                                    Time = 3,
                                })
                                task.wait(0.5)
                            else
                                if myRoot.Parent then
                                    myRoot.CFrame = returnCFrame
                                end
                            end
                        end
                    end

                    task.wait(COOLDOWN)
                end
            end)
        end,
    })

    UtilityGroup:AddToggle("T_DestroyFoodHoldables", {
        Text = "Destroy Food & Holdables",
        Default = false,
        Callback = function(v)
            destroyFoodActive = v and true or false
            if not destroyFoodActive then return end
            task.spawn(function()
                local items = {
                    FoodHamburger=true, FoodCoconut=true, FoodPizzaCheese=true, FoodPizzaPepperoni=true,
                    FoodHotdog=true, FoodMushroomPoison=true, FoodBread=true, FoodDippyEgg=true,
                    FoodMayonnaise=true, FoodFrenchFries=true, FoodMeatStick=true, FoodDonut=true,
                    FoodCakePink=true, FoodBanana=true, FoodBroccoli=true, FoodSodaCan=true,
                    CupMugWhite=true, CupMugBrown=true, PoopPile=true, PoopPileSparkle=true,
                    InstrumentGuitarBanjo=true, InstrumentGuitarViolin=true, InstrumentGuitarUkulele=true,
                    InstrumentWoodwindSaxophone=true, InstrumentWoodwindOcarina=true, InstrumentBrassVuvuzelaQwizik=true,
                    InstrumentBrassTrumpet=true, InstrumentDrumBongos=true, InstrumentDrumSnare=true,
                    InstrumentPianoMelodica=true, InstrumentVoiceMicrophone=true,
                }

                while destroyFoodActive do
                    local target = getPrimaryTarget()
                    local targetChar = target and target.Character
                    local targetRoot = targetChar and targetChar:FindFirstChild("HumanoidRootPart")
                    local folder = target and TWorkspace:FindFirstChild(target.Name .. "SpawnedInToys")
                    if target and target.Parent and targetRoot and folder then
                        for _, toy in ipairs(folder:GetChildren()) do
                            if items[toy.Name] and toy:IsA("Model") then
                                local holdPart = toy:FindFirstChild("HoldPart")
                                if holdPart then
                                    local held = false
                                    for _, desc in ipairs(toy:GetDescendants()) do
                                        if desc:IsA("WeldConstraint") or desc:IsA("Weld") then
                                            if (desc.Part0 and desc.Part0:IsDescendantOf(targetChar)) or (desc.Part1 and desc.Part1:IsDescendantOf(targetChar)) then
                                                held = true
                                                break
                                            end
                                        end
                                    end
                                    if not held and (holdPart.Position - targetRoot.Position).Magnitude < 8 then held = true end
                                    if not held then
                                        for _, part in ipairs(toy:GetDescendants()) do
                                            if part:IsA("BasePart") and (part.Position - targetRoot.Position).Magnitude < 8 then
                                                held = true
                                                break
                                            end
                                        end
                                    end
                                    if held then
                                        local grab = holdPart:FindFirstChild("HoldItemRemoteFunction")
                                        local drop = holdPart:FindFirstChild("DropItemRemoteFunction")
                                        if grab and drop then
                                            pcall(function()
                                                local char = LocalPlayer.Character
                                                if char then
                                                    grab:InvokeServer(toy, char)
                                                    drop:InvokeServer(toy, CFrame.new(0, -50000, 0), Vector3.zero)
                                                end
                                            end)
                                        end
                                    end
                                end
                            end
                        end
                    end
                    task.wait(0.05)
                end
            end)
        end,
    })

      UtilityGroup:AddToggle("T_RemoveAntiKickSticky", {
        Text = "Remove/Destroy Anti Kick / Sticky",
        Default = false,
        Callback = function(v)
            removeAntiKickStickyActive = v and true or false
            if not removeAntiKickStickyActive then return end

            task.spawn(function()
                local GrabEvents   = TReplicatedStorage:FindFirstChild("GrabEvents")
                local SetNetOwner  = GrabEvents and GrabEvents:FindFirstChild("SetNetworkOwner")
                local PlayerEvents = TReplicatedStorage:FindFirstChild("PlayerEvents")
                local StickyEvent  = PlayerEvents and PlayerEvents:FindFirstChild("StickyPartEvent")
                local LocalPlayer  = LocalPlayer

                local function sno(part)
                    if SetNetOwner and part then
                        pcall(function() SetNetOwner:FireServer(part, part.CFrame) end)
                    end
                end

                local function CheckNetworkOwnerOnPart(part)
                    local owner = part:FindFirstChild("PartOwner")
                    return owner and owner.Value == LocalPlayer.Name
                end

                local function GetMagnitude(part1, part2)
                    if not (part1 and part2) then return math.huge end
                    return (part1.Position - part2.Position).Magnitude
                end

                while removeAntiKickStickyActive do
                    task.wait()

                    local targets = GetSelectedTargets()
                    if #targets == 0 then
                        task.wait(0.25)
                        continue
                    end

                    local myChar     = LocalPlayer.Character
                    local myHRP      = myChar and myChar:FindFirstChild("HumanoidRootPart")
                    local myFirePart = myHRP and myHRP:FindFirstChild("FirePlayerPart")

                    for _, TargetPLR in ipairs(targets) do
                        if not removeAntiKickStickyActive then break end
                        if not TargetPLR or not TargetPLR.Parent then continue end

                        local TargetInv = TWorkspace:FindFirstChild(TargetPLR.Name .. "SpawnedInToys")
                        if not TargetInv then continue end

                        local Root = TargetPLR.Character and TargetPLR.Character:FindFirstChild("HumanoidRootPart")
                        local FirePlayerPart = Root and Root:FindFirstChild("FirePlayerPart")
                        if not FirePlayerPart then continue end

                        for _, v in pairs(TargetInv:GetChildren()) do
                            local StickyPart = v:FindFirstChild("StickyPart")
                            if StickyPart then
                                local StickyWeld = StickyPart:FindFirstChild("StickyWeld")
                                if not StickyWeld then continue end

                                if StickyPart.CanQuery then
                                    task.spawn(function()
                                        for _, part in pairs(v:GetChildren()) do
                                            if part:IsA("BasePart") then
                                                part.CanCollide = false
                                                part.CanTouch   = false
                                                part.CanQuery   = false
                                            end
                                        end
                                    end)
                                end

                                if myFirePart and StickyWeld.Part1 == myFirePart and GetMagnitude(FirePlayerPart, StickyPart) > 7 then
                                    continue
                                elseif not CheckNetworkOwnerOnPart(StickyPart) then
                                    sno(StickyPart)
                                else
                                    if StickyEvent then
                                        pcall(function()
                                            StickyEvent:FireServer(
                                                StickyPart,
                                                FirePlayerPart,
                                                CFrame.new(0, -10, 5)
                                            )
                                        end)
                                    end
                                end
                            end
                        end
                    end
                end
            end)
        end,
    })

    -- ================================================================
    -- LOOP SNOWBALL
    -- Spawns a snowball and re-explodes it on the target's root every
    -- cycle. Works on all selected targets in parallel.
    -- ================================================================
    local loopSnowballActive = false

    UtilityGroup:AddToggle("T_LoopSnowball", {
        Text = "Loop Snowball",
        Default = false,
        Tooltip = "Spawns a snowball and continuously explodes it on the selected targets",
        Callback = function(Value)
            loopSnowballActive = Value and true or false
            if not loopSnowballActive then return end

            task.spawn(function()
                local Remotes = {
                    SpawnToy    = TReplicatedStorage:FindFirstChild("MenuToys") and TReplicatedStorage.MenuToys:FindFirstChild("SpawnToyRemoteFunction"),
                    SetNetOwner = TReplicatedStorage:FindFirstChild("GrabEvents") and TReplicatedStorage.GrabEvents:FindFirstChild("SetNetworkOwner"),
                    BombExplode = TReplicatedStorage:FindFirstChild("BombEvents") and TReplicatedStorage.BombEvents:FindFirstChild("BombExplode"),
                }

                if not Remotes.SpawnToy or not Remotes.SetNetOwner or not Remotes.BombExplode then
                    warn("[Echo] Loop Snowball: required remotes not found")
                    loopSnowballActive = false
                    return
                end

                while loopSnowballActive do
                    local targets = GetSelectedTargets()
                    if #targets == 0 then
                        task.wait(0.25)
                        continue
                    end

                    local char = LocalPlayer.Character
                    local hrp  = char and char:FindFirstChild("HumanoidRootPart")
                    if not hrp then
                        task.wait(0.1)
                        continue
                    end

                    local inv = Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                    if not inv then
                        task.wait(0.1)
                        continue
                    end

                    local ball = inv:FindFirstChild("BallSnowball")

                    if not ball then
                        -- Spawn a fresh snowball
                        task.spawn(function()
                            pcall(function()
                                Remotes.SpawnToy:InvokeServer(
                                    "BallSnowball",
                                    hrp.CFrame * CFrame.new(0, 10, 20),
                                    Vector3.zero
                                )
                            end)
                        end)
                        task.wait(0.15)
                    else
                        local SoundPart = ball:FindFirstChild("SoundPart")

                        if SoundPart then
                            -- Grab one target per cycle, rotate through the list
                            -- so multi-target support doesn't spam a single root.
                            local target = targets[1]
                            if not target or not target.Character then
                                task.wait(0.1)
                                continue
                            end

                            local tRoot = target.Character:FindFirstChild("HumanoidRootPart")
                            if not tRoot then
                                task.wait(0.1)
                                continue
                            end

                            pcall(function()
                                Remotes.SetNetOwner:FireServer(SoundPart, SoundPart.CFrame)
                            end)
                            task.wait(0.05)

                            SoundPart.CFrame = tRoot.CFrame
                            task.wait(0.05)

                            local payload = {
                                Radius                 = 0,
                                Color                  = Color3.new(0, 0, 0),
                                TimeLength             = 0,
                                Model                  = ball,
                                Type                   = "SnowPoof",
                                ExplodesByFire         = false,
                                MaxForcePerStudSquared = 0,
                                Hitbox                 = SoundPart,
                                ImpactSpeed            = 0,
                                ExplodesByPointy       = false,
                                DestroysModel          = true,
                                PositionPart           = SoundPart,
                            }

                            pcall(function()
                                Remotes.BombExplode:FireServer(payload, Vector3.zero)
                            end)

                            task.wait(0.15)
                        else
                            task.wait(0.1)
                        end
                    end
                end

                -- Cleanup: destroy any leftover snowball when the toggle is off.
                pcall(function()
                    local inv = Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                    if inv then
                        for _, toy in ipairs(inv:GetChildren()) do
                            if toy.Name == "BallSnowball" then
                                local menu = TReplicatedStorage:FindFirstChild("MenuToys")
                                local destroy = menu and menu:FindFirstChild("DestroyToy")
                                if destroy then destroy:FireServer(toy) end
                            end
                        end
                    end
                end)
            end)
        end,
    })

    -- ================================================================
    -- LOOP BANANA RAGDOLL
    -- Holds a banana against the target's Left Leg via AlignPosition,
    -- which forces the server's ragdoll check to fire every frame.
    -- ================================================================
    local loopBananaRagdollActive = false

    UtilityGroup:AddToggle("T_LoopBananaRagdoll", {
        Text = "Loop Banana Ragdoll",
        Default = false,
        Tooltip = "Attaches a banana to the target's Left Leg to force ragdoll",
        Callback = function(Value)
            loopBananaRagdollActive = Value and true or false
            if not loopBananaRagdollActive then return end

            task.spawn(function()
                local function FWD(parent, part, time)
                    return parent:FindFirstChild(part) or parent:WaitForChild(part, time or 5)
                end

                local function CFP(parent, part)
                    return parent:FindFirstChild(part) ~= nil
                end

                local function sno(part)
                    pcall(function()
                        TReplicatedStorage.GrabEvents.SetNetworkOwner:FireServer(part, part.CFrame)
                    end)
                end

                local function unsno(part)
                    pcall(function()
                        TReplicatedStorage.GrabEvents.DestroyGrabLine:FireServer(part)
                    end)
                end

                local function CheckNetworkOwnerOnPart(Part)
                    return CFP(Part, "PartOwner") and Part:FindFirstChild("PartOwner").Value == LocalPlayer.Name
                end

                local function SpawnToy(ToyName)
                    local InPlot       = LocalPlayer:FindFirstChild("InPlot")
                    local InOwnedPlot  = LocalPlayer:FindFirstChild("InOwnedPlot")
                    local CanSpawnToy  = LocalPlayer:FindFirstChild("CanSpawnToy")

                    if InPlot and InPlot.Value and InOwnedPlot and not InOwnedPlot.Value then
                        InPlot:GetPropertyChangedSignal("Value"):Wait()
                    end
                    if CanSpawnToy and not CanSpawnToy.Value then
                        CanSpawnToy:GetPropertyChangedSignal("Value"):Wait()
                    end

                    local currentHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if not currentHRP then return nil end

                    local SpawnCF   = currentHRP.CFrame * CFrame.new(0, 14, 20)
                    local Container = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                    if not Container then return nil end

                    local spawnedObject = nil
                    local connection = Container.ChildAdded:Connect(function(child)
                        if child.Name == ToyName then
                            spawnedObject = child
                        end
                    end)

                    task.spawn(function()
                        pcall(function()
                            TReplicatedStorage.MenuToys.SpawnToyRemoteFunction:InvokeServer(ToyName, SpawnCF, Vector3.zero)
                        end)
                    end)

                    local start = tick()
                    repeat task.wait() until spawnedObject or (tick() - start) > 2.5

                    if connection then connection:Disconnect() end
                    return spawnedObject
                end

                local etc = {}
                local AlignPos
                local AtachNew

                while loopBananaRagdollActive do
                    task.wait()

                    local targets = GetSelectedTargets()
                    if #targets == 0 then
                        task.wait(0.25)
                        continue
                    end

                    local target = targets[1]

                    etc.Root = target and target.Character and target.Character:FindFirstChild("Left Leg")
                    if not etc.Root then continue end

                    local inv = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                    if not inv then continue end

                    local banana    = inv:FindFirstChild("FoodBanana")
                    local SoundPart = banana and banana:FindFirstChild("SoundPart")

                    if not SoundPart then
                        -- Destroy stale bananas before spawning a fresh one.
                        for _, v in pairs(inv:GetChildren()) do
                            if v.Name == "FoodBanana" then
                                pcall(function() TReplicatedStorage.MenuToys.DestroyToy:FireServer(v) end)
                            end
                        end

                        banana = SpawnToy("FoodBanana")
                        if not banana then continue end

                        SoundPart = FWD(banana, "SoundPart", 5)
                        if not SoundPart then continue end

                        local holdPart = FWD(banana, "HoldPart", 5)
                        if holdPart then
                            local holdRemote = FWD(holdPart, "HoldItemRemoteFunction", 5)
                            if holdRemote then
                                pcall(function() holdRemote:InvokeServer(banana, LocalPlayer.Character) end)
                            end
                        end

                        -- Wait for edible removal before dropping.
                        while CFP(banana, "EdiblePart") and loopBananaRagdollActive do task.wait() end

                        if holdPart then
                            local dropRemote = FWD(holdPart, "DropItemRemoteFunction", 5)
                            if dropRemote and LocalPlayer.Character then
                                pcall(function()
                                    dropRemote:InvokeServer(
                                        banana,
                                        LocalPlayer.Character:GetPivot() * CFrame.new(0, 15, -10),
                                        Vector3.zero
                                    )
                                end)
                            end
                        end

                        -- Secure network ownership of SoundPart.
                        repeat
                            task.wait(0.01)
                            SoundPart = banana and banana:FindFirstChild("SoundPart")
                            if not SoundPart then break end
                            sno(SoundPart)
                        until not SoundPart or CFP(SoundPart, "PartOwner") or not loopBananaRagdollActive

                        unsno(SoundPart)

                        local Atach = Instance.new("Attachment")
                        Atach.Parent = SoundPart

                        AlignPos = Instance.new("AlignPosition")
                        AlignPos.Responsiveness = 100
                        AlignPos.Parent = SoundPart
                        AlignPos.Attachment0 = Atach
                    end

                    -- Ownership drift check — if the server stole it, respawn.
                    for _, v in pairs(banana:GetChildren()) do
                        if CFP(v, "PartOwner") and not CheckNetworkOwnerOnPart(v) then
                            pcall(function() TReplicatedStorage.MenuToys.DestroyToy:FireServer(banana) end)
                            banana = nil
                            break
                        end
                    end

                    if not banana then continue end
                    AlignPos = SoundPart:FindFirstChild("AlignPosition")

                    if not AlignPos then
                        pcall(function() TReplicatedStorage.MenuToys.DestroyToy:FireServer(banana) end)
                        banana = nil
                        continue
                    end

                    AtachNew = etc.Root and etc.Root:FindFirstChild("LeftFootAttachment")
                    if not AtachNew then continue end
                    AlignPos.Attachment1 = AtachNew
                end

                -- Cleanup on stop.
                pcall(function()
                    local inv = TWorkspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
                    local banana = inv and inv:FindFirstChild("FoodBanana")
                    if banana then
                        local SoundPart = banana:FindFirstChild("SoundPart")
                        local AlignPos  = SoundPart and SoundPart:FindFirstChild("AlignPosition")
                        if AlignPos then AlignPos:Destroy() end
                        TReplicatedStorage.MenuToys.DestroyToy:FireServer(banana)
                    end
                end)
            end)
        end,
    })
    
    UtilityGroup:AddButton({
        Text = "Teleport to Target",
        Func = function()
            local targets = GetSelectedTargets()
            if #targets == 0 then return end
            local target = targets[1]
            if not target or not target.Character then return end
            local targetHRP = target.Character:FindFirstChild("HumanoidRootPart")
            if not targetHRP then return end
            local localChar = LocalPlayer.Character
            if not localChar then return end
            local localHRP = localChar:FindFirstChild("HumanoidRootPart")
            if not localHRP then return end
            localHRP.CFrame = targetHRP.CFrame + Vector3.new(0, 3, 0)
            localHRP.AssemblyLinearVelocity = Vector3.zero
            localHRP.AssemblyAngularVelocity = Vector3.zero
        end,
    })

    TPlayers.PlayerAdded:Connect(function()
        task.wait(0.5)
        TargetDropdown:SetValues(GetPlayerList())
    end)
    TPlayers.PlayerRemoving:Connect(function(plr)
        task.wait(0.5)
        TargetDropdown:SetValues(GetPlayerList())

        local entry = NonBlobTasks[plr.Name]
        if entry and entry.stop and entry.state then
            pcall(entry.stop, entry.state)
            NonBlobTasks[plr.Name] = nil
        end
        entry = BlobTasks[plr.Name]
        if entry and entry.stop and entry.state then
            pcall(entry.stop, entry.state)
            BlobTasks[plr.Name] = nil
        end
    end)

    pcall(function()
        Library:OnUnload(function()
            removeGucciActive = false
            destroyFoodActive = false
            removeAntiKickStickyActive = false
            TargetState.AutoSitBlobmanActive = false
            PalletRagdoll.Stop()
            if TargetState.AutoSitBlobmanThread then
                pcall(task.cancel, TargetState.AutoSitBlobmanThread)
                TargetState.AutoSitBlobmanThread = nil
            end
            TargetState.NonBlobActive = false
            TargetState.BlobActive = false
            StopAllNonBlob()
            StopAllBlob()
            -- Full cleanup of Blobman Kill (OP) shared state
            BlobKillOP.StopAll()
            BlobmanKillCore.Stop()
            if BlobmanKillCore.SelectedConnection then
                pcall(function() BlobmanKillCore.SelectedConnection:Disconnect() end)
                BlobmanKillCore.SelectedConnection = nil
            end
            if BlobmanKillCore.SelectedTask then
                pcall(task.cancel, BlobmanKillCore.SelectedTask)
                BlobmanKillCore.SelectedTask = nil
            end
            if BlobmanKillCore.AllTask then
                pcall(task.cancel, BlobmanKillCore.AllTask)
                BlobmanKillCore.AllTask = nil
            end
            ActiveNonBlob = nil
            ActiveBlob = nil
        end)
    end)
end

--endtarget tab

--visuals tab
-- VISUALS TAB
-- ============================================================
do
    local VisualsTab = Tabs.Visuals
    local VisualPlayers = Players
    local VisualWorkspace = Workspace


    local VisualState = {
        PCLD = false,
        PlayerPCLD = false,
        PCLDTransparency = 0.35,
        PlayerPCLDTransparency = 0.35,
        PCLDOutlineTransparency = 0,
        PlayerPCLDOutlineTransparency = 0,
        PCLDSmoothness = 0.8,
        PCLDBoxColor = EchoColorFromRGB(255, 255, 255),
        PCLDOutlineColor = EchoColorFromRGB(0, 0, 0),
        PlayerPCLDBoxColor = EchoColorFromRGB(0, 170, 255),
        PlayerPCLDOutlineColor = EchoColorFromRGB(255, 255, 255),
        PlayerESP = false,
        PlayerFillColor = EchoColorFromRGB(255, 0, 0),
        PlayerFillTransparency = 0.5,
        PlayerOutlineColor = EchoColorFromRGB(255, 255, 255),
        PlayerOutlineTransparency = 0,
        PlayerDepthMode = "AlwaysOnTop",
        NameESP = false,
        NameColor = EchoColorFromRGB(255, 255, 255),
        NameFont = "GothamBold",
        NameSize = 16,
        NameStroke = true,
        StickyESP = false,
        StickyFillColor = EchoColorFromRGB(255, 230, 0),
        StickyOutlineColor = EchoColorFromRGB(255, 255, 255),
        StickyTransparency = 0.25,
        StickyOutlineTransparency = 0,
        BlobmanESP = false,
        BlobmanFillColor = EchoColorFromRGB(0, 170, 255),
        BlobmanOutlineColor = EchoColorFromRGB(255, 255, 255),
        BlobmanTransparency = 0.35,
        BlobmanOutlineTransparency = 0,
        RealisticWater = false,
        WaterSplash = false,
    }

    local PlayerHighlights = {}
    local NameTags = {}
    local PCLDVisuals = {}
    local PlayerPCLDVisuals = {}
    local StickyVisuals = {}
    local BlobmanVisuals = {}
    local VisualConnections = {}

    local ESPFolder = Instance.new("Folder")
    ESPFolder.Name = "EchoVisuals"
    ESPFolder.Parent = VisualWorkspace

    function destroyMap(map)
        for key, object in pairs(map) do
            pcall(function()
                if type(object) == "table" then
                    if object.ServerAnchor then object.ServerAnchor:Destroy() end
                    if object.PlayerAnchor then object.PlayerAnchor:Destroy() end
                    if object.Anchor then object.Anchor:Destroy() end
                    if object.ServerBox then object.ServerBox:Destroy() end
                    if object.PlayerBox then object.PlayerBox:Destroy() end
                    if object.Box then object.Box:Destroy() end
                else
                    object:Destroy()
                end
            end)
            map[key] = nil
        end
    end

    function makeHighlight(store, key, adornee, name, fill, fillTransparency, outline, outlineTransparency, depthMode)
        local h = store[key]
        if not h or not h.Parent then
            h = Instance.new("Highlight")
            h.Name = name
            h.Parent = ESPFolder
            store[key] = h
        end
        h.Adornee = adornee
        h.FillColor = fill
        h.FillTransparency = fillTransparency
        h.OutlineColor = outline
        h.OutlineTransparency = outlineTransparency
        h.DepthMode = depthMode or Enum.HighlightDepthMode.AlwaysOnTop
        h.Enabled = true
        return h
    end

    function clearPlayerESP(player)
        local h = PlayerHighlights[player]
        if h then
            pcall(function() h:Destroy() end)
            PlayerHighlights[player] = nil
        end
    end

    function updatePlayerESP(player)
        if player == LocalPlayer or not VisualState.PlayerESP then
            clearPlayerESP(player)
            return
        end

        local character = player.Character
        if not character or not character.Parent then
            clearPlayerESP(player)
            return
        end

        local depth = VisualState.PlayerDepthMode == "Occluded" and Enum.HighlightDepthMode.Occluded or Enum.HighlightDepthMode.AlwaysOnTop

        local h = PlayerHighlights[player]
        if not h or not h.Parent then
            h = Instance.new("Highlight")
            h.Name = "Echo_PlayerESP"
            h.Parent = character
            PlayerHighlights[player] = h
        elseif h.Parent ~= character then
            h.Parent = character
        end

        h.Adornee = character
        h.FillColor = VisualState.PlayerFillColor
        h.FillTransparency = VisualState.PlayerFillTransparency
        h.OutlineColor = VisualState.PlayerOutlineColor
        h.OutlineTransparency = VisualState.PlayerOutlineTransparency
        h.DepthMode = depth
        h.Enabled = true
    end

    function refreshPlayerESP()
        if not VisualState.PlayerESP then
            destroyMap(PlayerHighlights)
            return
        end
        for _, player in ipairs(VisualPlayers:GetPlayers()) do
            updatePlayerESP(player)
        end
    end

    local HeadshotCache = {}

    function getNameESPHeadshot(player)
        if HeadshotCache[player.UserId] then
            return HeadshotCache[player.UserId]
        end

        local image = ""
        pcall(function()
            image = VisualPlayers:GetUserThumbnailAsync(player.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size150x150)
        end)

        HeadshotCache[player.UserId] = image or ""
        return HeadshotCache[player.UserId]
    end

    function clearNameESP(player)
        local gui = NameTags[player]
        if gui then
            pcall(function() gui:Destroy() end)
            NameTags[player] = nil
        end
    end

    function updateNameESP(player)
        if player == LocalPlayer or not VisualState.NameESP then
            clearNameESP(player)
            return
        end

        local character = player.Character
        local head = character and character:FindFirstChild("Head")
        local root = character and character:FindFirstChild("HumanoidRootPart")

        if not head or not root then
            clearNameESP(player)
            return
        end

        local gui = NameTags[player]
        if not gui or not gui.Parent then
            gui = Instance.new("BillboardGui")
            gui.Name = "Echo_NameESP"
            gui.Size = UDim2.fromOffset(190, 52)
            gui.StudsOffset = Vector3.new(0, 3.5, 0)
            gui.AlwaysOnTop = true
            gui.MaxDistance = 10000
            gui.LightInfluence = 0
            gui.Parent = ESPFolder

            local avatar = Instance.new("ImageLabel")
            avatar.Name = "Avatar"
            avatar.Size = UDim2.fromOffset(44, 44)
            avatar.Position = UDim2.fromOffset(0, 4)
            avatar.BackgroundColor3 = EchoColorFromRGB(20, 20, 20)
            avatar.BackgroundTransparency = 0.15
            avatar.BorderSizePixel = 0
            avatar.ScaleType = Enum.ScaleType.Crop
            avatar.Image = getNameESPHeadshot(player)
            avatar.Parent = gui

            local avatarCorner = Instance.new("UICorner")
            avatarCorner.CornerRadius = UDim.new(0, 6)
            avatarCorner.Parent = avatar

            local avatarStroke = Instance.new("UIStroke")
            avatarStroke.Thickness = 2
            avatarStroke.Color = EchoColorFromRGB(255, 255, 255)
            avatarStroke.Transparency = 0.35
            avatarStroke.Parent = avatar

            local nameLabel = Instance.new("TextLabel")
            nameLabel.Name = "Name"
            nameLabel.BackgroundTransparency = 1
            nameLabel.Size = UDim2.new(1, -52, 0, 26)
            nameLabel.Position = UDim2.fromOffset(52, 3)
            nameLabel.TextXAlignment = Enum.TextXAlignment.Left
            nameLabel.TextYAlignment = Enum.TextYAlignment.Center
            nameLabel.TextStrokeColor3 = EchoColorNew(0, 0, 0)
            nameLabel.Parent = gui

            local distanceLabel = Instance.new("TextLabel")
            distanceLabel.Name = "Distance"
            distanceLabel.BackgroundTransparency = 1
            distanceLabel.Size = UDim2.new(1, -52, 0, 18)
            distanceLabel.Position = UDim2.fromOffset(52, 28)
            distanceLabel.TextXAlignment = Enum.TextXAlignment.Left
            distanceLabel.TextYAlignment = Enum.TextYAlignment.Center
            distanceLabel.TextColor3 = EchoColorFromRGB(255, 205, 110)
            distanceLabel.TextSize = 14
            distanceLabel.Font = Enum.Font.GothamMedium
            distanceLabel.TextStrokeColor3 = EchoColorNew(0, 0, 0)
            distanceLabel.TextStrokeTransparency = 0.4
            distanceLabel.Parent = gui

            NameTags[player] = gui
        end

        gui.Adornee = head

        local nameLabel = gui:FindFirstChild("Name")
        local distanceLabel = gui:FindFirstChild("Distance")

        if nameLabel then
            nameLabel.Text = player.Name
            nameLabel.TextColor3 = VisualState.NameColor
            nameLabel.TextSize = VisualState.NameSize
            nameLabel.Font = Enum.Font[VisualState.NameFont] or Enum.Font.GothamBold
            nameLabel.TextStrokeTransparency = VisualState.NameStroke and 0.35 or 1
        end

        if distanceLabel then
            local camera = Workspace.CurrentCamera
            if camera then
                local distance = (camera.CFrame.Position - root.Position).Magnitude
                distanceLabel.Text = tostring(math.floor(distance + 0.5)) .. "m"
            else
                distanceLabel.Text = "--m"
            end
        end
    end

    function clearAllNameESP()
        destroyMap(NameTags)

        local roots = {VisualWorkspace, LocalPlayer:FindFirstChildOfClass("PlayerGui")}
        pcall(function() table.insert(roots, CoreGui) end)

        for _, root in ipairs(roots) do
            if root then
                for _, object in ipairs(root:GetDescendants()) do
                    if object:IsA("BillboardGui") and object.Name == "Echo_NameESP" then
                        pcall(function()
                            object.Enabled = false
                            object:Destroy()
                        end)
                    end
                end
            end
        end
    end

    function refreshNameESP()
        if not VisualState.NameESP then
            clearAllNameESP()
            return
        end

        for _, player in ipairs(VisualPlayers:GetPlayers()) do
            updateNameESP(player)
        end
    end

    function isPCLD(object)
        return object and object.Name == "PlayerCharacterLocationDetector" and object:IsA("BasePart")
    end

    local function createPCLDAnchor(name, size, cf, parent)
        local anchor = Instance.new("Part")
        anchor.Name = name
        anchor.Anchored = true
        anchor.CanCollide = false
        anchor.CanTouch = false
        anchor.CanQuery = false
        anchor.Transparency = 1
        anchor.Size = size
        anchor.CFrame = cf
        anchor.Parent = parent
        return anchor
    end

    local function createPCLDBox(name, anchor)
        local box = Instance.new("SelectionBox")
        box.Name = name
        box.Adornee = anchor
        box.LineThickness = 0.01
        box.Parent = anchor
        return box
    end

    -- Server PCLD is completely separate from Player PCLD.
    -- Player PCLD is keyed directly by Player, so it can never jump to another player's detector.
    function clearPCLD(object)
        local data = PCLDVisuals[object]
        if data then
            pcall(function()
                if data.ServerAnchor then data.ServerAnchor:Destroy() end
                if data.ServerBox then data.ServerBox:Destroy() end
            end)
            PCLDVisuals[object] = nil
        end
    end

    function updatePCLD(object)
        if not VisualState.PCLD or not isPCLD(object) or not object.Parent then
            clearPCLD(object)
            return
        end

        local data = PCLDVisuals[object]
        if not data or not data.ServerAnchor or not data.ServerAnchor.Parent then
            local serverAnchor = createPCLDAnchor("Echo_PCLD_Server_Anchor", object.Size, object.CFrame, ESPFolder)
            local serverBox = createPCLDBox("Echo_PCLD_Server", serverAnchor)
            data = {
                ServerAnchor = serverAnchor,
                ServerBox = serverBox,
            }
            PCLDVisuals[object] = data
        end

        local serverAnchor = data.ServerAnchor
        local serverBox = data.ServerBox

        serverBox.Adornee = serverAnchor
        serverBox.Color3 = VisualState.PCLDOutlineColor
        serverBox.SurfaceColor3 = VisualState.PCLDBoxColor
        serverBox.SurfaceTransparency = VisualState.PCLDTransparency
        serverBox.Transparency = VisualState.PCLDOutlineTransparency
        if serverBox:IsA("Highlight") then
            serverBox.Enabled = VisualState.PCLD
        else
            serverBox.Visible = VisualState.PCLD
        end

        pcall(function()
            serverAnchor.Size = object.Size
            serverAnchor.CFrame = object.CFrame
        end)
    end

    local function clearPlayerPCLD(player)
        local data = PlayerPCLDVisuals[player]
        if data then
            pcall(function()
                if data.Anchor then data.Anchor:Destroy() end
                if data.Box then data.Box:Destroy() end
            end)
            PlayerPCLDVisuals[player] = nil
        end
    end

    local function updatePlayerPCLD(player)
        if not VisualState.PlayerPCLD or not player then
            clearPlayerPCLD(player)
            return
        end

        local character = player.Character
        local hrp = character and character:FindFirstChild("HumanoidRootPart")
        if not hrp or not hrp.Parent then
            clearPlayerPCLD(player)
            return
        end

        local data = PlayerPCLDVisuals[player]
        if not data or not data.Anchor or not data.Anchor.Parent then
            local anchor = createPCLDAnchor("Echo_PCLD_Player_Anchor_" .. tostring(player.UserId), hrp.Size, hrp.CFrame, ESPFolder)
            local box = createPCLDBox("Echo_PCLD_Player_" .. tostring(player.UserId), anchor)
            data = {
                Anchor = anchor,
                Box = box,
            }
            PlayerPCLDVisuals[player] = data
        end

        data.Box.Adornee = data.Anchor
        data.Box.Color3 = VisualState.PlayerPCLDOutlineColor
        data.Box.SurfaceColor3 = VisualState.PlayerPCLDBoxColor
        data.Box.SurfaceTransparency = VisualState.PlayerPCLDTransparency
        data.Box.Transparency = VisualState.PlayerPCLDOutlineTransparency
        if data.Box:IsA("Highlight") then
            data.Box.Enabled = VisualState.PlayerPCLD
        else
            data.Box.Visible = VisualState.PlayerPCLD
        end

        pcall(function()
            data.Anchor.Size = hrp.Size
        end)
    end

    local function refreshPlayerPCLD()
        if not VisualState.PlayerPCLD then
            for player in pairs(PlayerPCLDVisuals) do
                clearPlayerPCLD(player)
            end
            return
        end

        for _, player in ipairs(VisualPlayers:GetPlayers()) do
            updatePlayerPCLD(player)
        end
    end

    function updatePCLDTracking()
        if VisualState.PCLD then
            local smooth = math.clamp(VisualState.PCLDSmoothness, 0, 1)
            local alpha = 1 - (smooth * 0.92)

            for object, data in pairs(PCLDVisuals) do
                if not object or not object.Parent or not isPCLD(object) or not data then
                    clearPCLD(object)
                else
                    local serverAnchor = data.ServerAnchor
                    if not serverAnchor or not serverAnchor.Parent then
                        clearPCLD(object)
                    else
                        pcall(function()
                            serverAnchor.Size = object.Size
                            serverAnchor.CFrame = serverAnchor.CFrame:Lerp(object.CFrame, alpha)
                        end)
                    end
                end
            end
        elseif next(PCLDVisuals) then
            for object in pairs(PCLDVisuals) do
                clearPCLD(object)
            end
        end

        if VisualState.PlayerPCLD then
            local smooth = math.clamp(VisualState.PCLDSmoothness, 0, 1)
            local alpha = 1 - (smooth * 0.92)

            for _, player in ipairs(VisualPlayers:GetPlayers()) do
                if player then
                    local character = player.Character
                    local hrp = character and character:FindFirstChild("HumanoidRootPart")
                    local data = PlayerPCLDVisuals[player]

                    if hrp and hrp.Parent then
                        if not data or not data.Anchor or not data.Anchor.Parent then
                            updatePlayerPCLD(player)
                            data = PlayerPCLDVisuals[player]
                        end

                        if data and data.Anchor and data.Anchor.Parent then
                            pcall(function()
                                data.Anchor.Size = hrp.Size
                                data.Anchor.CFrame = data.Anchor.CFrame:Lerp(hrp.CFrame, alpha)
                            end)
                            if data.Box:IsA("Highlight") then
                                data.Box.Enabled = true
                            else
                                data.Box.Visible = true
                            end
                        end
                    else
                        clearPlayerPCLD(player)
                    end
                end
            end
        elseif next(PlayerPCLDVisuals) then
            for player in pairs(PlayerPCLDVisuals) do
                clearPlayerPCLD(player)
            end
        end
    end

    function refreshPCLD()
        -- Server PCLD only scans server detector objects.
        if VisualState.PCLD then
            for _, object in ipairs(VisualWorkspace:GetChildren()) do
                if isPCLD(object) then updatePCLD(object) end
            end

            for _, object in ipairs(VisualWorkspace:GetDescendants()) do
                if isPCLD(object) then updatePCLD(object) end
            end
        else
            for object in pairs(PCLDVisuals) do
                clearPCLD(object)
            end
        end

        -- Player PCLD never uses PlayerCharacterLocationDetector matching.
        refreshPlayerPCLD()
    end

    function isStickyObject(object)
        if not object then return false end

        -- Normal sticky toys.
        if object:IsA("Model") then
            -- Anti Kick toys are renamed to AntiKick by Echo immediately after
            -- spawning. They must be considered sticky ESP targets even while
            -- the server is still creating/replicating StickyPart or StickyWeld.
            if object.Name == "AntiKick" then
                return true
            end

            if object:FindFirstChild("StickyPart", true) then
                return true
            end

            return false
        end

        -- Also support games/executor states where the sticky toy is exposed
        -- as a BasePart instead of the complete Model.
        if object:IsA("BasePart") and object.Name == "StickyPart" then
            local ancestor = object.Parent
            while ancestor and ancestor ~= VisualWorkspace do
                if ancestor:IsA("Model") then
                    if ancestor.Name == "AntiKick" or ancestor:FindFirstChild("StickyPart", true) then
                        return true
                    end
                    break
                end
                ancestor = ancestor.Parent
            end
            return true
        end

        return false
    end

    function isBlobman(object)
        return object:IsA("Model") and object.Name == "CreatureBlobman"
    end

    function refreshSpecial(enabled, store, predicate, visualName, fill, fillTransparency, outline, outlineTransparency)
        if not enabled then
            destroyMap(store)
            return
        end

        local seen = {}
        outlineTransparency = outlineTransparency or 0

        for _, object in ipairs(VisualWorkspace:GetDescendants()) do
            if predicate(object) then
                seen[object] = true
                makeHighlight(store, object, object, visualName, fill, fillTransparency, outline, outlineTransparency, Enum.HighlightDepthMode.AlwaysOnTop)
            end
        end

        for object, h in pairs(store) do
            if not seen[object] or not object.Parent then
                pcall(function() h:Destroy() end)
                store[object] = nil
            end
        end
    end

    local WaterSplashFolder = Instance.new("Folder")
    WaterSplashFolder.Name = "EchoWaterSplashes"
    WaterSplashFolder.Parent = VisualWorkspace

    local WaterSplashTracked = {}
    local WaterSplashLastSample = 0
    local WaterSplashScanInterval = 1 / 15
    local WaterSplashCooldown = 0.18
    local WaterSplashMaxEffects = 28
    local WaterSplashOceanBounds = nil
    local WaterSplashOceanBoundsTime = 0
    local WaterSplashEffectCount = 0
    local WaterSplashAddedConnection = nil
    local WaterSplashRemovingConnection = nil

    local function waterSplashIsIgnored(part)
        if not part or not part:IsA("BasePart") then return true end
        if part:IsDescendantOf(ESPFolder) or part:IsDescendantOf(WaterSplashFolder) then return true end
        return false
    end

    local function waterSplashRefreshOceanBounds()
        local now = os.clock()
        if WaterSplashOceanBounds and now - WaterSplashOceanBoundsTime < 2 then
            return WaterSplashOceanBounds
        end

        local ocean = getXOCUOceanModel()
        if not ocean then
            WaterSplashOceanBounds = nil
            WaterSplashOceanBoundsTime = now
            return nil
        end

        local minX, maxX, minY, maxY, minZ, maxZ
        for _, part in ipairs(ocean:GetChildren()) do
            if part:IsA("BasePart") and part.Parent then
                local half = part.Size * 0.5
                local corners = {
                    part.CFrame:PointToWorldSpace(Vector3.new(-half.X, -half.Y, -half.Z)),
                    part.CFrame:PointToWorldSpace(Vector3.new( half.X,  half.Y,  half.Z)),
                }
                local a, b = corners[1], corners[2]
                minX = math.min(minX or a.X, b.X)
                maxX = math.max(maxX or a.X, b.X)
                minY = math.min(minY or a.Y, b.Y)
                maxY = math.max(maxY or a.Y, b.Y)
                minZ = math.min(minZ or a.Z, b.Z)
                maxZ = math.max(maxZ or a.Z, b.Z)
            end
        end

        if minX then
            WaterSplashOceanBounds = {
                minX = minX, maxX = maxX, minY = minY, maxY = maxY,
                minZ = minZ, maxZ = maxZ,
                surfaceY = maxY,
            }
        else
            WaterSplashOceanBounds = nil
        end
        WaterSplashOceanBoundsTime = now
        return WaterSplashOceanBounds
    end

    local function waterSplashIsWaterAt(position)
        local bounds = waterSplashRefreshOceanBounds()
        if bounds and position.X >= bounds.minX - 1 and position.X <= bounds.maxX + 1
            and position.Z >= bounds.minZ - 1 and position.Z <= bounds.maxZ + 1
            and position.Y <= bounds.surfaceY + 1.5 and position.Y >= bounds.minY - 2 then
            return true
        end

        local terrain = VisualWorkspace.Terrain
        if terrain then
            local params = RaycastParams.new()
            params.FilterType = Enum.RaycastFilterType.Exclude
            params.FilterDescendantsInstances = {ESPFolder, WaterSplashFolder}
            local result = VisualWorkspace:Raycast(position + Vector3.new(0, 1.5, 0), Vector3.new(0, -4, 0), params)
            if result and result.Material == Enum.Material.Water then
                return true
            end
        end
        return false
    end

    local function waterSplashNearWater(part)
        local bounds = waterSplashRefreshOceanBounds()
        if not bounds then return true end
        local p = part.Position
        local half = part.Size * 0.5
        return p.X + half.X >= bounds.minX - 2
            and p.X - half.X <= bounds.maxX + 2
            and p.Z + half.Z >= bounds.minZ - 2
            and p.Z - half.Z <= bounds.maxZ + 2
            and p.Y - half.Y <= bounds.surfaceY + 3
            and p.Y + half.Y >= bounds.minY - 3
    end

    local function waterSplashImpactPoint(part)
        local size = part.Size
        local bounds = waterSplashRefreshOceanBounds()
        if bounds then
            local bottom = part.CFrame:PointToWorldSpace(Vector3.new(0, -size.Y * 0.5, 0))
            if bottom.X >= bounds.minX - 2 and bottom.X <= bounds.maxX + 2
                and bottom.Z >= bounds.minZ - 2 and bottom.Z <= bounds.maxZ + 2
                and bottom.Y <= bounds.surfaceY + 1.5 and bottom.Y >= bounds.minY - 2 then
                return Vector3.new(bottom.X, bounds.surfaceY + 0.03, bottom.Z)
            end
        end

        local bottom = part.CFrame:PointToWorldSpace(Vector3.new(0, -size.Y * 0.5, 0))
        if waterSplashIsWaterAt(bottom) then return bottom end
        return nil
    end

    local function waterSplashEffect(position, velocity)
        if not VisualState.WaterSplash then return end
        if WaterSplashEffectCount >= WaterSplashMaxEffects then return end
        WaterSplashEffectCount = WaterSplashEffectCount + 1

        local impactSpeed = math.clamp(velocity.Magnitude, 0.5, 180)
        local scale = math.clamp(0.55 + impactSpeed / 55, 0.55, 2.8)
        local folder = WaterSplashFolder
        local created = {}
        local splashColor = Color3.fromRGB(190, 225, 235)
        pcall(function()
            local terrain = VisualWorkspace.Terrain
            if terrain and terrain.WaterColor then
                splashColor = terrain.WaterColor:Lerp(Color3.new(1, 1, 1), 0.18)
            end
        end)

        local function makePart(name, shape, size, cf, transparency, material)
            local part = Instance.new("Part")
            part.Name = name
            part.Shape = shape or Enum.PartType.Block
            part.Size = size
            part.CFrame = cf
            part.Anchored = true
            part.CanCollide = false
            part.CanQuery = false
            part.CanTouch = false
            part.CastShadow = false
            part.Material = material or Enum.Material.SmoothPlastic
            part.Color = splashColor
            part.Transparency = transparency
            part.Parent = folder
            created[#created + 1] = part
            return part
        end

        -- Impact depression and concentric surface ripples.
        local holder = makePart("Impact", Enum.PartType.Ball, Vector3.new(0.18,0.18,0.18), CFrame.new(position), 1)

        local ring = makePart("RippleInner", Enum.PartType.Cylinder,
            Vector3.new(0.035, 0.7, 0.7) * scale,
            CFrame.new(position + Vector3.new(0, 0.025, 0)), 0.08)
        ring.Material = Enum.Material.SmoothPlastic
        TweenService:Create(ring, TweenInfo.new(0.32, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Size = Vector3.new(0.025, 5.5, 5.5) * scale, Transparency = 1,
        }):Play()

        local ring2 = makePart("RippleMid", Enum.PartType.Cylinder,
            Vector3.new(0.025, 0.5, 0.5) * scale,
            CFrame.new(position + Vector3.new(0, 0.035, 0)), 0.28)
        ring2.Material = Enum.Material.SmoothPlastic
        TweenService:Create(ring2, TweenInfo.new(0.62, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Size = Vector3.new(0.018, 11, 11) * scale, Transparency = 1,
        }):Play()

        local ring3 = makePart("RippleOuter", Enum.PartType.Cylinder,
            Vector3.new(0.018, 0.4, 0.4) * scale,
            CFrame.new(position + Vector3.new(0, 0.045, 0)), 0.48)
        ring3.Material = Enum.Material.SmoothPlastic
        TweenService:Create(ring3, TweenInfo.new(1.05, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Size = Vector3.new(0.012, 19, 19) * scale, Transparency = 1,
        }):Play()

        -- Low, thin foam sheet spreading away from the impact.
        local foam = makePart("FoamSheet", Enum.PartType.Cylinder,
            Vector3.new(0.06, 0.8, 0.8) * scale,
            CFrame.new(position + Vector3.new(0,0.055,0)), 0.3)
        TweenService:Create(foam, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Size = Vector3.new(0.035, 7.5, 7.5) * scale, Transparency = 1,
        }):Play()

        -- Central crown. Height is driven by impact speed, so tiny drops stay tiny.
        local crownHeight = math.clamp(0.35 + impactSpeed * 0.045, 0.35, 7.5)
        local crownCount = math.clamp(math.floor(6 + impactSpeed / 14), 6, 14)
        for i = 1, crownCount do
            local angle = (math.pi * 2) * (i / crownCount) + math.random() * 0.15
            local radius = (0.22 + math.random() * 0.75) * scale
            local height = crownHeight * (0.45 + math.random() * 0.8)
            local width = (0.035 + math.random() * 0.075) * scale
            local x = math.cos(angle) * radius
            local z = math.sin(angle) * radius
            local column = makePart("CrownColumn", Enum.PartType.Cylinder,
                Vector3.new(width, math.max(0.08,height), width),
                CFrame.new(position + Vector3.new(x, height * 0.5, z)), 0.18)
            local target = column.CFrame + Vector3.new(x * 0.8, 0.2 + height * 0.28, z * 0.8)
            TweenService:Create(column, TweenInfo.new(0.28 + math.random()*0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                Size = Vector3.new(width * 0.2, math.max(0.03,height * 0.18), width * 0.2),
                Transparency = 1, CFrame = target,
            }):Play()
        end

        -- Main droplets and fine mist. Every impact gets a few, even at very low speed.
        local droplets = math.clamp(math.floor(4 + impactSpeed / 5), 4, 18)
        for i = 1, droplets do
            local angle = math.random() * math.pi * 2
            local radial = math.clamp(0.25 + impactSpeed * 0.025, 0.25, 4.5) * (0.55 + math.random()*0.7) * scale
            local upward = math.clamp(1.2 + impactSpeed * 0.065, 1.2, 11) * (0.65 + math.random()*0.7)
            local size = math.clamp(0.025 + impactSpeed * 0.0007, 0.025, 0.09) * (0.7 + math.random()*0.7)
            local drop = makePart("Droplet", Enum.PartType.Ball,
                Vector3.new(size,size,size), CFrame.new(position + Vector3.new(0,0.1,0)), 0.12)
            local travel = Vector3.new(math.cos(angle)*radial, upward, math.sin(angle)*radial)
            local life = 0.35 + math.random()*0.5
            TweenService:Create(drop, TweenInfo.new(life, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
                CFrame = drop.CFrame + travel,
                Size = Vector3.new(size*0.45,size*0.45,size*0.45),
                Transparency = 1,
            }):Play()
        end

        local mistCount = math.clamp(math.floor(3 + impactSpeed / 15), 3, 7)
        for i = 1, mistCount do
            local puff = makePart("Mist", Enum.PartType.Ball,
                Vector3.new(0.06,0.06,0.06) * scale,
                CFrame.new(position + Vector3.new((math.random()-0.5)*1.2,0.15,(math.random()-0.5)*1.2)),
                0.6)
            TweenService:Create(puff, TweenInfo.new(0.55 + math.random()*0.3, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {
                Size = Vector3.new(0.22,0.22,0.22) * scale,
                CFrame = puff.CFrame + Vector3.new((math.random()-0.5)*2, 0.5 + math.random()*1.6, (math.random()-0.5)*2),
                Transparency = 1,
            }):Play()
        end

        task.delay(1.25, function()
            WaterSplashEffectCount = math.max(0, WaterSplashEffectCount - 1)
            for _, object in ipairs(created) do
                pcall(function()
                    if object and object.Parent then object:Destroy() end
                end)
            end
        end)
    end

    local function waterSplashCheckPart(part, now)
        if not VisualState.WaterSplash or waterSplashIsIgnored(part) then return end
        if not part.Parent or part.Anchored then
            WaterSplashTracked[part] = nil
            return
        end

        local position = part.Position
        local velocity = part.AssemblyLinearVelocity
        local data = WaterSplashTracked[part]
        if not data then
            WaterSplashTracked[part] = {position = position, cframe = part.CFrame, lastSplash = -math.huge}
            return
        end

        local previous = data.position
        local previousCFrame = data.cframe or part.CFrame
        data.position = position
        data.cframe = part.CFrame

        local moved = (position - previous).Magnitude
        if moved < 0.01 and velocity.Magnitude < 0.05 then return end
        if now - data.lastSplash < WaterSplashCooldown then return end
        if not waterSplashNearWater(part) then return end

        local size = part.Size
        local previousBottom = previousCFrame:PointToWorldSpace(Vector3.new(0, -size.Y * 0.5, 0))
        local impactPoint = waterSplashImpactPoint(part)
        if not impactPoint then return end

        local currentBottom = part.CFrame:PointToWorldSpace(Vector3.new(0, -size.Y * 0.5, 0))
        local wasWater = waterSplashIsWaterAt(previousBottom)
        local isWater = waterSplashIsWaterAt(currentBottom) or waterSplashIsWaterAt(position)
        local crossedSurface = (not wasWater) and isWater

        if crossedSurface and velocity.Y <= 1 then
            data.lastSplash = now
            waterSplashEffect(impactPoint, velocity)
        end
    end

    local function trackWaterSplashPart(part)
        if waterSplashIsIgnored(part) then return end
        if not WaterSplashTracked[part] then
            WaterSplashTracked[part] = {
                position = part.Position,
                cframe = part.CFrame,
                lastSplash = -math.huge,
            }
        end
    end

    local function untrackWaterSplashPart(part)
        WaterSplashTracked[part] = nil
    end

    local function startWaterSplashTracking()
        if WaterSplashAddedConnection then return end

        table.clear(WaterSplashTracked)
        for _, part in ipairs(VisualWorkspace:GetDescendants()) do
            if part:IsA("BasePart") and not waterSplashIsIgnored(part) then
                trackWaterSplashPart(part)
            end
        end

        WaterSplashAddedConnection = VisualWorkspace.DescendantAdded:Connect(function(object)
            if object:IsA("BasePart") then
                trackWaterSplashPart(object)
            end
        end)

        WaterSplashRemovingConnection = VisualWorkspace.DescendantRemoving:Connect(function(object)
            if object:IsA("BasePart") then
                untrackWaterSplashPart(object)
            end
        end)
    end

    local function stopWaterSplashTracking()
        if WaterSplashAddedConnection then
            pcall(function() WaterSplashAddedConnection:Disconnect() end)
            WaterSplashAddedConnection = nil
        end
        if WaterSplashRemovingConnection then
            pcall(function() WaterSplashRemovingConnection:Disconnect() end)
            WaterSplashRemovingConnection = nil
        end
        table.clear(WaterSplashTracked)
    end

    local function clearWaterSplashes()
        stopWaterSplashTracking()
        WaterSplashEffectCount = 0
        for _, object in ipairs(WaterSplashFolder:GetChildren()) do
            pcall(function() object:Destroy() end)
        end
    end

    local function updateWaterSplash()
        if not VisualState.WaterSplash then
            if WaterSplashAddedConnection then stopWaterSplashTracking() end
            return
        end

        if not WaterSplashAddedConnection then startWaterSplashTracking() end

        local now = os.clock()
        if now - WaterSplashLastSample < WaterSplashScanInterval then return end
        WaterSplashLastSample = now

        for part in pairs(WaterSplashTracked) do
            if not part.Parent then
                WaterSplashTracked[part] = nil
            else
                waterSplashCheckPart(part, now)
            end
        end
    end

    local WaterBackup = {active = false, parts = {}}

    function getXOCUOceanModel()
        local map = VisualWorkspace:FindFirstChild("Map")
        local always = map and map:FindFirstChild("AlwaysHereTweenedObjects")
        local ocean = always and always:FindFirstChild("Ocean")
        local object = ocean and ocean:FindFirstChild("Object")
        return object and object:FindFirstChild("ObjectModel")
    end

    function enableRealisticWater()
        if WaterBackup.active then return end

        local model = getXOCUOceanModel()
        if not model then
            warn("[Echo] XOCU water: FTAP ocean path was not found.")
            return
        end

        local terrain = VisualWorkspace.Terrain

        for _, part in ipairs(model:GetChildren()) do
            if part:IsA("BasePart") then
                local size = part.Size
                local cf = part.CFrame
                local region = Region3.new(cf.Position - size / 2, cf.Position + size / 2):ExpandToGrid(4)

                local ok, materials, occupancy = pcall(function()
                    return terrain:ReadVoxels(region, 4)
                end)

                table.insert(WaterBackup.parts, {
                    clone = part:Clone(),
                    parent = part.Parent,
                    region = region,
                    materials = ok and materials or nil,
                    occupancy = ok and occupancy or nil,
                })

                terrain:FillRegion(region, 4, Enum.Material.Water)
                part:Destroy()
            end
        end

        WaterBackup.active = true
    end

    function disableRealisticWater()
        if not WaterBackup.active then return end

        local model = getXOCUOceanModel()

        for _, item in ipairs(WaterBackup.parts) do
            if item.materials and item.occupancy then
                pcall(function()
                    VisualWorkspace.Terrain:WriteVoxels(item.region, 4, item.materials, item.occupancy)
                end)
            end

            if item.clone and not item.clone.Parent then
                pcall(function()
                    item.clone.Parent = model or item.parent
                end)
            end
        end

        WaterBackup.parts = {}
        WaterBackup.active = false
    end

    function hookPlayer(player)
        if player == LocalPlayer then return end

        local key = "Character_" .. player.UserId
        if VisualConnections[key] then
            VisualConnections[key]:Disconnect()
        end

        VisualConnections[key] = player.CharacterAdded:Connect(function()
            task.wait(0.35)
            clearPlayerESP(player)
            clearNameESP(player)
            updatePlayerESP(player)
            updateNameESP(player)
        end)

        task.defer(function()
            updatePlayerESP(player)
            updateNameESP(player)
        end)
    end

    for _, player in ipairs(VisualPlayers:GetPlayers()) do
        hookPlayer(player)
    end

    VisualConnections.PlayerAdded = VisualPlayers.PlayerAdded:Connect(hookPlayer)

    VisualConnections.PlayerRemoving = VisualPlayers.PlayerRemoving:Connect(function(player)
        clearPlayerESP(player)
        clearNameESP(player)

        local key = "Character_" .. player.UserId
        if VisualConnections[key] then
            VisualConnections[key]:Disconnect()
            VisualConnections[key] = nil
        end
    end)

    VisualConnections.LocalRespawn = LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.5)
        refreshPlayerESP()
        refreshNameESP()
    end)

    VisualConnections.ChildAdded = VisualWorkspace.ChildAdded:Connect(function(object)
        if VisualState.PCLD and isPCLD(object) then
            task.defer(function()
                if object and object.Parent then updatePCLD(object) end
            end)
        end
    end)

    VisualConnections.DescAdded = VisualWorkspace.DescendantAdded:Connect(function(object)
        if VisualState.PCLD and isPCLD(object) then
            task.defer(function()
                if object and object.Parent then updatePCLD(object) end
            end)
        end
    end)

    VisualConnections.DescRemoving = VisualWorkspace.DescendantRemoving:Connect(function(object)
        if PCLDVisuals[object] then clearPCLD(object) end
    end)

    VisualConnections.Render = RunService.RenderStepped:Connect(function()
        updatePCLDTracking()
        updateWaterSplash()
        if VisualState.PlayerESP then refreshPlayerESP() end
        if VisualState.NameESP then refreshNameESP() end
    end)

    task.spawn(function()
        while not Library.Unloaded do
            refreshSpecial(VisualState.StickyESP, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
            refreshSpecial(VisualState.BlobmanESP, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
            task.wait(0.2)
        end
    end)


    -- ESP LAYOUT
    -- The four requested ESP options are kept together so the Visuals tab is easier to use.
    local ESPBox = VisualsTab:AddLeftTabbox("ESP")
    local ESPToggles = ESPBox:AddTab("ESP", "scan")
    local ESPSettings = ESPBox:AddTab("Config", "settings")
    local VisualTabBox = VisualsTab:AddLeftTabbox("Visuals")
    local BlackholeTab = VisualTabBox:AddTab("Blackhole", "galaxy")
    local LineVisualGroup = VisualTabBox:AddTab("Line customization", "line-squiggle")

    local PlayerToggle = ESPToggles:AddToggle("EchoPlayerESP", {
        Text = "Player ESP",
        Default = false,
        Callback = function(v)
            VisualState.PlayerESP = v and true or false
            refreshPlayerESP()
        end,
    })
    PlayerToggle:AddColorPicker("EchoPlayerQuickColor", {
        Default = VisualState.PlayerFillColor,
        Title = "Player ESP Colour",
        Callback = function(v)
            VisualState.PlayerFillColor = v
            refreshPlayerESP()
        end,
    })
    PlayerToggle:AddColorPicker("EchoPlayerQuickOutline", {
        Default = VisualState.PlayerOutlineColor,
        Title = "Player Outline",
        Callback = function(v)
            VisualState.PlayerOutlineColor = v
            refreshPlayerESP()
        end,
    })

    local ServerESPToggle = ESPToggles:AddToggle("EchoServerESP", {
        Text = "Server ESP [PCLD]",
        Default = false,
        Callback = function(v)
            VisualState.PCLD = v and true or false
            refreshPCLD()
        end,
    })
    ServerESPToggle:AddColorPicker("EchoServerQuickColor", {
        Default = VisualState.PCLDBoxColor,
        Title = "PCLD Box Colour",
        Callback = function(v)
            VisualState.PCLDBoxColor = v
            refreshPCLD()
        end,
    })
    ServerESPToggle:AddColorPicker("EchoServerQuickOutline", {
        Default = VisualState.PCLDOutlineColor,
        Title = "PCLD Outline",
        Callback = function(v)
            VisualState.PCLDOutlineColor = v
            refreshPCLD()
        end,
    })

    local PlayerPCLDToggle = ESPToggles:AddToggle("EchoPlayerPCLD", {
        Text = "Player ESP [PCLD]",
        Default = false,
        Callback = function(v)
            VisualState.PlayerPCLD = v and true or false
            refreshPCLD()
        end,
    })
    PlayerPCLDToggle:AddColorPicker("EchoPlayerPCLDQuickColor", {
        Default = VisualState.PlayerPCLDBoxColor,
        Title = "Player PCLD Colour",
        Callback = function(v)
            VisualState.PlayerPCLDBoxColor = v
            refreshPCLD()
        end,
    })
    PlayerPCLDToggle:AddColorPicker("EchoPlayerPCLDQuickOutline", {
        Default = VisualState.PlayerPCLDOutlineColor,
        Title = "Player PCLD Outline",
        Callback = function(v)
            VisualState.PlayerPCLDOutlineColor = v
            refreshPCLD()
        end,
    })

    local NameToggle = ESPToggles:AddToggle("EchoNameESP", {
        Text = "Name ESP",
        Default = false,
        Callback = function(v)
            VisualState.NameESP = v and true or false
            if not VisualState.NameESP then
                clearAllNameESP()
            else
                refreshNameESP()
            end
        end,
    })
    NameToggle:AddColorPicker("EchoNameQuickColor", {
        Default = VisualState.NameColor,
        Title = "Name Colour",
        Callback = function(v)
            VisualState.NameColor = v
            refreshNameESP()
        end,
    })

    local StickyToggle = ESPToggles:AddToggle("EchoStickyESP", {
        Text = "Sticky ESP",
        Default = false,
        Callback = function(v)
            VisualState.StickyESP = v and true or false
            if not VisualState.StickyESP then
                destroyMap(StickyVisuals)
            else
                -- Refresh immediately so freshly spawned AntiKick toys are
                -- highlighted without waiting for the periodic ESP pass.
                refreshSpecial(
                    true,
                    StickyVisuals,
                    isStickyObject,
                    "Echo_StickyESP",
                    VisualState.StickyFillColor,
                    VisualState.StickyTransparency,
                    VisualState.StickyOutlineColor,
                    VisualState.StickyOutlineTransparency
                )
            end
        end,
    })
    StickyToggle:AddColorPicker("EchoStickyQuickColor", {
        Default = VisualState.StickyFillColor,
        Title = "Sticky Fill Colour",
        Callback = function(v)
            VisualState.StickyFillColor = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    })
    StickyToggle:AddColorPicker("EchoStickyQuickOutline", {
        Default = VisualState.StickyOutlineColor,
        Title = "Sticky Outline Colour",
        Callback = function(v)
            VisualState.StickyOutlineColor = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    })

    local BlobmanToggle = ESPToggles:AddToggle("EchoBlobmanESP", {
        Text = "Blobman ESP",
        Default = false,
        Callback = function(v)
            VisualState.BlobmanESP = v and true or false
            if not VisualState.BlobmanESP then
                destroyMap(BlobmanVisuals)
            else
                refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
            end
        end,
    })
    BlobmanToggle:AddColorPicker("EchoBlobQuickColor", {
        Default = VisualState.BlobmanFillColor,
        Title = "Blobman Fill Colour",
        Callback = function(v)
            VisualState.BlobmanFillColor = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    })
    BlobmanToggle:AddColorPicker("EchoBlobQuickOutline", {
        Default = VisualState.BlobmanOutlineColor,
        Title = "Blobman Outline",
        Callback = function(v)
            VisualState.BlobmanOutlineColor = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    })

    -- Settings uses the same Configure style as the reference UI.
    local ConfigureLabel = ESPSettings:AddLabel("Configure")
    local setSettingsVisible
    local ConfigureDropdown = ESPSettings:AddDropdown("EchoESPConfigure", {
        Text = "Configure",
        Values = {"Player ESP", "PCLD View", "Player ESP [PCLD]", "Name ESP", "Sticky ESP", "Blobman ESP"},
        Default = "Player ESP",
        Callback = function(v)
            if setSettingsVisible then
                setSettingsVisible(v)
            end
        end,
    })
    LineVisualGroup:AddDivider()

    local PlayerSettingsElements = {}
    local PCLDSettingsElements = {}
    local PlayerPCLDSettingsElements = {}
    local NameSettingsElements = {}
    local StickySettingsElements = {}
    local BlobmanSettingsElements = {}

    function addSetting(list, element)
        table.insert(list, element)
        return element
    end

    addSetting(PlayerSettingsElements, ESPSettings:AddLabel("PLAYER ESP"))
    addSetting(PlayerSettingsElements, ESPSettings:AddDropdown("EchoPlayerDepth", {
        Text = "Highlight Mode",
        Values = {"AlwaysOnTop", "Occluded"},
        Default = "AlwaysOnTop",
        Callback = function(v)
            VisualState.PlayerDepthMode = v
            refreshPlayerESP()
        end,
    }))
    addSetting(PlayerSettingsElements, ESPSettings:AddSlider("EchoPlayerTransparency", {
        Text = "Fill Transparency",
        Default = 0.5,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PlayerFillTransparency = v
            refreshPlayerESP()
        end,
    }))
    addSetting(PlayerSettingsElements, ESPSettings:AddSlider("EchoPlayerOutlineTransparency", {
        Text = "Outline Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PlayerOutlineTransparency = v
            refreshPlayerESP()
        end,
    }))
    addSetting(PlayerSettingsElements, ESPSettings:AddLabel("Fill Colour"):AddColorPicker("EchoPlayerFill", {
        Default = VisualState.PlayerFillColor,
        Title = "Fill Colour",
        Callback = function(v)
            VisualState.PlayerFillColor = v
            refreshPlayerESP()
        end,
    }))
    addSetting(PlayerSettingsElements, ESPSettings:AddLabel("Outline Colour"):AddColorPicker("EchoPlayerOutline", {
        Default = VisualState.PlayerOutlineColor,
        Title = "Outline Colour",
        Callback = function(v)
            VisualState.PlayerOutlineColor = v
            refreshPlayerESP()
        end,
    }))

    addSetting(PCLDSettingsElements, ESPSettings:AddLabel("PCLD VIEW"))
    addSetting(PCLDSettingsElements, ESPSettings:AddSlider("EchoPCLDTransparency", {
        Text = "Transparency",
        Default = 0.35,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PCLDTransparency = v
            refreshPCLD()
        end,
    }))
    addSetting(PCLDSettingsElements, ESPSettings:AddSlider("EchoPCLDOutlineTransparency", {
        Text = "Outline Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PCLDOutlineTransparency = v
            refreshPCLD()
        end,
    }))
    addSetting(PCLDSettingsElements, ESPSettings:AddSlider("EchoPCLDSmoothness", {
        Text = "Smooth Tracking",
        Default = 0.8,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PCLDSmoothness = v
        end,
    }))
    addSetting(PCLDSettingsElements, ESPSettings:AddLabel("Box Colour"):AddColorPicker("EchoPCLDBoxColor", {
        Default = VisualState.PCLDBoxColor,
        Title = "PCLD Box Colour",
        Callback = function(v)
            VisualState.PCLDBoxColor = v
            refreshPCLD()
        end,
    }))
    addSetting(PCLDSettingsElements, ESPSettings:AddLabel("Outline Colour"):AddColorPicker("EchoPCLDOutlineColor", {
        Default = VisualState.PCLDOutlineColor,
        Title = "PCLD Outline Colour",
        Callback = function(v)
            VisualState.PCLDOutlineColor = v
            refreshPCLD()
        end,
    }))

    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddLabel("PLAYER ESP (PCLD)"))
    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddSlider("EchoPlayerPCLDTransparency", {
        Text = "Fill Transparency",
        Default = 0.35,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PlayerPCLDTransparency = v
            refreshPCLD()
        end,
    }))
    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddSlider("EchoPlayerPCLDOutlineTransparency", {
        Text = "Outline Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PlayerPCLDOutlineTransparency = v
            refreshPCLD()
        end,
    }))
    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddSlider("EchoPlayerPCLDSmoothness", {
        Text = "Smooth Tracking",
        Default = 0.8,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.PCLDSmoothness = v
        end,
    }))
    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddLabel("Fill Colour"):AddColorPicker("EchoPlayerPCLDFillColor", {
        Default = VisualState.PlayerPCLDBoxColor,
        Title = "Player PCLD Fill Colour",
        Callback = function(v)
            VisualState.PlayerPCLDBoxColor = v
            refreshPCLD()
        end,
    }))
    addSetting(PlayerPCLDSettingsElements, ESPSettings:AddLabel("Outline Colour"):AddColorPicker("EchoPlayerPCLDOutlineColor", {
        Default = VisualState.PlayerPCLDOutlineColor,
        Title = "Player PCLD Outline Colour",
        Callback = function(v)
            VisualState.PlayerPCLDOutlineColor = v
            refreshPCLD()
        end,
    }))

    addSetting(NameSettingsElements, ESPSettings:AddLabel("NAME ESP"))
    addSetting(NameSettingsElements, ESPSettings:AddDropdown("EchoNameFont", {
        Text = "Font",
        Values = {"Gotham", "GothamBold", "GothamMedium", "Arial", "ArialBold", "SourceSans", "SourceSansBold", "Code", "Fantasy", "SciFi"},
        Default = "GothamBold",
        Callback = function(v)
            VisualState.NameFont = v
            refreshNameESP()
        end,
    }))
    addSetting(NameSettingsElements, ESPSettings:AddSlider("EchoNameSize", {
        Text = "Text Size",
        Default = 16,
        Min = 10,
        Max = 30,
        Rounding = 0,
        Callback = function(v)
            VisualState.NameSize = v
            refreshNameESP()
        end,
    }))
    addSetting(NameSettingsElements, ESPSettings:AddToggle("EchoNameStroke", {
        Text = "Text Stroke",
        Default = true,
        Callback = function(v)
            VisualState.NameStroke = v
            refreshNameESP()
        end,
    }))
    addSetting(NameSettingsElements, ESPSettings:AddLabel("Name Colour"):AddColorPicker("EchoNameColor", {
        Default = VisualState.NameColor,
        Title = "Name Colour",
        Callback = function(v)
            VisualState.NameColor = v
            refreshNameESP()
        end,
    }))

    addSetting(StickySettingsElements, ESPSettings:AddLabel("STICKY ESP"))
    addSetting(StickySettingsElements, ESPSettings:AddSlider("EchoStickyTransparency", {
        Text = "Transparency",
        Default = 0.25,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.StickyTransparency = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    }))
    addSetting(StickySettingsElements, ESPSettings:AddSlider("EchoStickyToyTransparency", {
        Text = "Sticky transparancy",
        Default = 1,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            EchoAntiKickToyTransparency = tonumber(v) or 0
            if EchoAntiKickCurrentToy and EchoAntiKickCurrentToy.Parent and EchoApplyAntiKickToyTransparency then
                EchoApplyAntiKickToyTransparency(EchoAntiKickCurrentToy)
            end
        end,
    }))
    addSetting(StickySettingsElements, ESPSettings:AddSlider("EchoStickyOutlineTransparency", {
        Text = "Outline Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.StickyOutlineTransparency = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    }))
    addSetting(StickySettingsElements, ESPSettings:AddLabel("Fill Colour"):AddColorPicker("EchoStickyFill", {
        Default = VisualState.StickyFillColor,
        Title = "Sticky Fill Colour",
        Callback = function(v)
            VisualState.StickyFillColor = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    }))
    addSetting(StickySettingsElements, ESPSettings:AddLabel("Outline Colour"):AddColorPicker("EchoStickyOutline", {
        Default = VisualState.StickyOutlineColor,
        Title = "Sticky Outline Colour",
        Callback = function(v)
            VisualState.StickyOutlineColor = v
            refreshSpecial(true, StickyVisuals, isStickyObject, "Echo_StickyESP", VisualState.StickyFillColor, VisualState.StickyTransparency, VisualState.StickyOutlineColor, VisualState.StickyOutlineTransparency)
        end,
    }))

    addSetting(BlobmanSettingsElements, ESPSettings:AddLabel("BLOBMAN ESP"))
    addSetting(BlobmanSettingsElements, ESPSettings:AddSlider("EchoBlobTransparency", {
        Text = "Transparency",
        Default = 0.35,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.BlobmanTransparency = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    }))
    addSetting(BlobmanSettingsElements, ESPSettings:AddSlider("EchoBlobOutlineTransparency", {
        Text = "Outline Transparency",
        Default = 0,
        Min = 0,
        Max = 1,
        Rounding = 2,
        Callback = function(v)
            VisualState.BlobmanOutlineTransparency = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    }))
    addSetting(BlobmanSettingsElements, ESPSettings:AddLabel("Fill Colour"):AddColorPicker("EchoBlobFill", {
        Default = VisualState.BlobmanFillColor,
        Title = "Blobman Fill Colour",
        Callback = function(v)
            VisualState.BlobmanFillColor = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    }))
    addSetting(BlobmanSettingsElements, ESPSettings:AddLabel("Outline Colour"):AddColorPicker("EchoBlobOutline", {
        Default = VisualState.BlobmanOutlineColor,
        Title = "Blobman Outline Colour",
        Callback = function(v)
            VisualState.BlobmanOutlineColor = v
            refreshSpecial(true, BlobmanVisuals, isBlobman, "Echo_BlobmanESP", VisualState.BlobmanFillColor, VisualState.BlobmanTransparency, VisualState.BlobmanOutlineColor, VisualState.BlobmanOutlineTransparency)
        end,
    }))

    function setSettingsVisible(active)
        local function apply(list, visible)
            for _, element in ipairs(list) do
                pcall(function() element:SetVisible(visible) end)
            end
        end
        apply(PlayerSettingsElements, active == "Player ESP")
        apply(PCLDSettingsElements, active == "PCLD View")
        apply(PlayerPCLDSettingsElements, active == "Player ESP [PCLD]")
        apply(NameSettingsElements, active == "Name ESP")
        apply(StickySettingsElements, active == "Sticky ESP")
        apply(BlobmanSettingsElements, active == "Blobman ESP")
    end

    ConfigureDropdown.Callback = function(v)
        setSettingsVisible(v)
    end

    setSettingsVisible("Player ESP")

    local CosmicBlackhole = {
        Enabled = false,
        Active = {},
        Connection = nil,
        Selected = "None",
        Options = {
            ["Echo Blackhole"] = {Id = "rbxassetid://108005971027516", TextureId = "rbxassetid://137783993203311", Scale = 0.75, Type = "echo"},
            ["EternityHole"] = {Id = "rbxassetid://99340233207059", Scale = 1.0, Type = "model"},
            ["Custom Blackhole"] = {Id = "rbxassetid://18882467235", Scale = 50.0, Type = "decal"},
        },
    }
    
    function WipeOriginal(bh)
        if not bh then return end
        for _, desc in ipairs(bh:GetDescendants()) do
            if desc:IsA("BasePart") or desc:IsA("MeshPart") or desc:IsA("UnionOperation") then
                desc.Transparency = 1
                desc.CastShadow = false
            elseif desc:IsA("ParticleEmitter") or desc:IsA("Trail") or desc:IsA("Beam") or desc:IsA("Smoke") or desc:IsA("Fire") or desc:IsA("Sparkles") then
                desc.Enabled = false
            elseif desc:IsA("PointLight") or desc:IsA("SpotLight") or desc:IsA("SurfaceLight") then
                desc.Enabled = false
            elseif desc:IsA("Decal") or desc:IsA("Texture") then
                desc.Transparency = 1
            elseif desc:IsA("BillboardGui") or desc:IsA("SurfaceGui") then
                desc.Enabled = false
            end
        end
        if bh:IsA("BasePart") then
            bh.Transparency = 1
        end
        if bh:IsA("Model") and bh.PrimaryPart then
            bh.PrimaryPart.Transparency = 1
        end
    end
    
    function RestoreOriginal(bh)
        if not bh then return end
        for _, desc in ipairs(bh:GetDescendants()) do
            if desc:IsA("BasePart") or desc:IsA("MeshPart") or desc:IsA("UnionOperation") then
                desc.Transparency = 0
            elseif desc:IsA("ParticleEmitter") or desc:IsA("Trail") or desc:IsA("Beam") or desc:IsA("Smoke") or desc:IsA("Fire") or desc:IsA("Sparkles") then
                desc.Enabled = true
            elseif desc:IsA("PointLight") or desc:IsA("SpotLight") or desc:IsA("SurfaceLight") then
                desc.Enabled = true
            elseif desc:IsA("Decal") or desc:IsA("Texture") then
                desc.Transparency = 0
            elseif desc:IsA("BillboardGui") or desc:IsA("SurfaceGui") then
                desc.Enabled = true
            end
        end
        if bh:IsA("BasePart") then
            bh.Transparency = 0
        end
        if bh:IsA("Model") and bh.PrimaryPart then
            bh.PrimaryPart.Transparency = 0
        end
    end
    
    function Cleanup(bh)
        if not bh then return end
        local data = CosmicBlackhole.Active[bh]
        if not data then return end
        if data.Anim then
            pcall(function() data.Anim:Disconnect() end)
        end
        if data.Model then
            pcall(function() data.Model:Destroy() end)
        end
        RestoreOriginal(bh)
        CosmicBlackhole.Active[bh] = nil
    end
    
    function Build(bh)
        if not bh or not bh.Parent then return end
        if CosmicBlackhole.Active[bh] then return end

        local hole, pos, option, data, assetId, model
        local cleanId, success, result, imageId, thumbUrl
        local decalPart, billboard, imageLabel, offset, firstPart, cf
        local currentPos, baseSize

        hole = bh:FindFirstChild("Hole") or bh:FindFirstChildWhichIsA("BasePart")
        if not hole then
            return
        end
        WipeOriginal(bh)
        pos = hole.Position
        option = CosmicBlackhole.Options[CosmicBlackhole.Selected] or CosmicBlackhole.Options["Echo Blackhole"]
        data = {
            Time = 0,
            Hole = hole,
            Anim = nil,
            Model = nil,
            SpinSpeed = 0,
            TargetSpin = 0.8,
            Scale = 0,
            TargetScale = option.Scale,
            StartTime = tick(),
        }
        assetId = option.Id
        model = nil
        if option.Type == "decal" then
            cleanId = tonumber((assetId:gsub("rbxassetid://", "")))
            success, result = pcall(function()
                return game:GetService("MarketplaceService"):GetProductInfo(cleanId)
            end)
            imageId = nil
            if success and result and result.IconImageAssetId and result.IconImageAssetId ~= 0 then
                imageId = result.IconImageAssetId
            end
            if not imageId then
                imageId = cleanId
            end
            thumbUrl = "rbxthumb://type=Asset&id=" .. tostring(imageId) .. "&w=420&h=420"
            decalPart = Instance.new("Part")
            decalPart.Anchored = true
            decalPart.CanCollide = false
            decalPart.CastShadow = false
            decalPart.Transparency = 1
            decalPart.Size = Vector3.new(1, 1, 1)
            decalPart.CFrame = CFrame.new(pos)
            decalPart.Parent = bh
            billboard = Instance.new("BillboardGui")
            billboard.Size = UDim2.new(option.Scale, 0, option.Scale, 0)
            billboard.AlwaysOnTop = true
            billboard.Parent = decalPart
            imageLabel = Instance.new("ImageLabel")
            imageLabel.Size = UDim2.new(1, 0, 1, 0)
            imageLabel.BackgroundTransparency = 1
            imageLabel.Image = thumbUrl
            imageLabel.Parent = billboard
            model = decalPart
        else
            success, result = pcall(function()
                return game:GetObjects(assetId)[1]
            end)
            if success and result then
                model = result
            end
            if not model then
                success, result = pcall(function()
                    local mp = Instance.new("MeshPart")
                    mp.MeshId = assetId
                    return mp
                end)
                if success and result then
                    model = result
                end
            end
        end
        if not model then
            RestoreOriginal(bh)
            return
        end
        model.Parent = bh
        model.Name = "CosmicMesh"
        for _, desc in ipairs(model:GetDescendants()) do
            if desc:IsA("AnimationController") or desc:IsA("Animator") or desc:IsA("Animation") then
                desc:Destroy()
            end
        end
        if CosmicBlackhole.Selected == "Echo Blackhole" then
            for _, desc in ipairs(model:GetDescendants()) do
                if desc:IsA("PointLight") or desc:IsA("SpotLight") or desc:IsA("SurfaceLight") then
                    desc:Destroy()
                elseif desc:IsA("ParticleEmitter") then
                    desc.LightEmission = 0
                    desc.Rate = desc.Rate * 0.2
                    if desc.Brightness then
                        desc.Brightness = desc.Brightness * 0.1
                    end
                elseif desc:IsA("Trail") or desc:IsA("Beam") then
                    desc.LightEmission = 0
                elseif desc:IsA("Fire") or desc:IsA("Sparkles") then
                    desc:Destroy()
                elseif desc:IsA("BasePart") or desc:IsA("MeshPart") then
                    if desc.Material == Enum.Material.Neon then
                        desc.Material = Enum.Material.SmoothPlastic
                    end
                end
            end
        end
        if option.Type == "echo" and option.TextureId then
            for _, desc in ipairs(model:GetDescendants()) do
                if desc:IsA("MeshPart") then
                    pcall(function() desc.TextureID = option.TextureId end)
                end
            end
            if model:IsA("MeshPart") then
                pcall(function() model.TextureID = option.TextureId end)
            end
        end
        if option.Type == "decal" then
            data.Model = model
            data.Anim = RunService.Heartbeat:Connect(function(dt)
                if not bh or not bh.Parent then
                    Cleanup(bh)
                    return
                end
                currentPos = hole.Position
                data.Time = data.Time + dt * data.SpinSpeed
                if data.Model then
                    data.Model.CFrame = CFrame.new(currentPos) * CFrame.Angles(0, data.Time, 0)
                    if not data.Model:GetAttribute("BaseSize") then
                        data.Model:SetAttribute("BaseSize", data.Model.Size)
                    end
                    baseSize = data.Model:GetAttribute("BaseSize")
                    data.Model.Size = baseSize * data.Scale
                    billboard = data.Model:FindFirstChildOfClass("BillboardGui")
                    if billboard then
                        billboard.Size = UDim2.new(data.Scale, 0, data.Scale, 0)
                    end
                end
            end)
        elseif model:IsA("Model") then
            for _, part in ipairs(model:GetDescendants()) do
                if part:IsA("BasePart") or part:IsA("MeshPart") then
                    part.Anchored = true
                    part.CanCollide = false
                    part.CastShadow = false
                    part.Size = part.Size * 0.01
                end
                if part:IsA("Smoke") or part:IsA("ParticleEmitter") then
                    if part.Name:lower():find("smog") or part.Name:lower():find("fog") or part.Name:lower():find("mist") or part.Name:lower():find("haze") or part.Name:lower():find("cloud") then
                        part:Destroy()
                    end
                end
            end
            if model.PrimaryPart then
                offset = pos - model.PrimaryPart.Position
                model:SetPrimaryPartCFrame(model.PrimaryPart.CFrame + offset)
            else
                firstPart = nil
                for _, part in ipairs(model:GetDescendants()) do
                    if part:IsA("BasePart") then
                        firstPart = part
                        break
                    end
                end
                if firstPart then
                    offset = pos - firstPart.Position
                    for _, part in ipairs(model:GetDescendants()) do
                        if part:IsA("BasePart") then
                            part.CFrame = part.CFrame + offset
                        end
                    end
                end
            end
            data.Model = model
            data.Anim = RunService.Heartbeat:Connect(function(dt)
                if not bh or not bh.Parent then
                    Cleanup(bh)
                    return
                end
                currentPos = hole.Position
                data.Time = data.Time + dt * data.SpinSpeed
                if data.Model then
                    if data.Model:IsA("Model") then
                        if data.Model.PrimaryPart then
                            cf = CFrame.new(currentPos) * CFrame.Angles(0, data.Time, 0)
                            data.Model:SetPrimaryPartCFrame(cf)
                        else
                            firstPart = nil
                            for _, part in ipairs(data.Model:GetDescendants()) do
                                if part:IsA("BasePart") then
                                    firstPart = part
                                    break
                                end
                            end
                            if firstPart then
                                for _, part in ipairs(data.Model:GetDescendants()) do
                                    if part:IsA("BasePart") then
                                        part.CFrame = CFrame.new(currentPos) * CFrame.Angles(0, data.Time, 0)
                                    end
                                end
                            end
                        end
                        for _, part in ipairs(data.Model:GetDescendants()) do
                            if part:IsA("BasePart") or part:IsA("MeshPart") then
                                if not part:GetAttribute("BaseSize") then
                                    part:SetAttribute("BaseSize", part.Size / 0.01)
                                end
                                baseSize = part:GetAttribute("BaseSize")
                                part.Size = baseSize * data.Scale
                            end
                        end
                    elseif data.Model:IsA("BasePart") then
                        data.Model.CFrame = CFrame.new(currentPos) * CFrame.Angles(0, data.Time, 0)
                        if not data.Model:GetAttribute("BaseSize") then
                            data.Model:SetAttribute("BaseSize", data.Model.Size / 0.01)
                        end
                        data.Model.Size = data.Model:GetAttribute("BaseSize") * data.Scale
                    end
                end
            end)
        elseif model:IsA("BasePart") then
            model.Anchored = true
            model.CanCollide = false
            model.CastShadow = false
            model.Size = model.Size * 0.01
            model.CFrame = CFrame.new(pos)
            data.Model = model
            data.Anim = RunService.Heartbeat:Connect(function(dt)
                if not bh or not bh.Parent then
                    Cleanup(bh)
                    return
                end
                currentPos = hole.Position
                data.Time = data.Time + dt * data.SpinSpeed
                if data.Model then
                    data.Model.CFrame = CFrame.new(currentPos) * CFrame.Angles(0, data.Time, 0)
                    if not data.Model:GetAttribute("BaseSize") then
                        data.Model:SetAttribute("BaseSize", data.Model.Size / 0.01)
                    end
                    data.Model.Size = data.Model:GetAttribute("BaseSize") * data.Scale
                end
            end)
        end
        bh.AncestryChanged:Connect(function()
            if not bh.Parent then
                Cleanup(bh)
            end
        end)
        CosmicBlackhole.Active[bh] = data
    end
    
    function StartSystem()
        if CosmicBlackhole.Connection then return end
        CosmicBlackhole.Connection = VisualWorkspace.ChildAdded:Connect(function(obj)
            if not CosmicBlackhole.Enabled then return end
            if obj.Name:lower():find("blackhole") then
                task.wait(0.2)
                Build(obj)
            end
        end)
    end

    local BlackholeEnabledToggle = BlackholeTab:AddToggle("EchoBlackholeVisuals", {
        Text = "Enable Blackhole Visuals",
        Default = false,
        Callback = function(v)
            CosmicBlackhole.Enabled = v and CosmicBlackhole.Selected ~= "None"
    
            if CosmicBlackhole.Enabled then
                StartSystem()
                for _, obj in ipairs(VisualWorkspace:GetChildren()) do
                    if obj.Name:lower():find("blackhole") then
                        Build(obj)
                    end
                end
            else
                if CosmicBlackhole.Connection then
                    pcall(function() CosmicBlackhole.Connection:Disconnect() end)
                    CosmicBlackhole.Connection = nil
                end
                for bh, _ in pairs(CosmicBlackhole.Active) do
                    Cleanup(bh)
                end
            end
        end,
    })
    
    BlackholeTab:AddDropdown("EchoBlackholeSelection", {
        Text = "Blackhole Visual",
        Values = {"None", "Echo Blackhole", "EternityHole", "Custom Blackhole"},
        Default = "None",
        Multi = false,
        Callback = function(val)
            CosmicBlackhole.Selected = val
    
            if CosmicBlackhole.Enabled then
                for bh, _ in pairs(CosmicBlackhole.Active) do
                    Cleanup(bh)
                end
                if val == "None" then
                    CosmicBlackhole.Enabled = false
                    if CosmicBlackhole.Connection then
                        pcall(function() CosmicBlackhole.Connection:Disconnect() end)
                        CosmicBlackhole.Connection = nil
                    end
                    pcall(function() BlackholeEnabledToggle:SetValue(false) end)
                    return
                end
                for _, obj in ipairs(VisualWorkspace:GetChildren()) do
                    if obj.Name:lower():find("blackhole") then
                        Build(obj)
                    end
                end
            end
        end,
    })
    

do
    -- GRAB LINE VISUALS
    -- FOV and time controls are intentionally omitted because Echo already provides those elsewhere in the Visuals tab.
    local LineState = {}
    local LineFns = {}

    LineState.LineSoundService = SoundService
    LineState.LineDebris = Debris
    LineState.LineRunService = RunService

    LineState.lineSelectedTexture = "Low Quality"
    LineState.lineCustomTextureId = ""
    LineState.lineUseSelectedTexture = true
    LineState.lineUseCustomTexture = false
    LineState.lineTextureSpeed = 1
    LineState.lineTextureLength = 1
    LineState.lineCustomGrabSoundId = ""
    LineState.lineSpeedEnabled = false
    LineState.lineLengthEnabled = false
    LineState.lineHeartbeatConnection = nil
    LineState.lineSyncAccumulator = 0
    LineState.lineToggleSyncing = false
    LineState.lineNotificationSoundId = "rbxassetid://97643101798871"


    getgenv().EchoLineTextureIds = {
        ["Low Quality"] = "",
        ["Non-Gamepass"] = "rbxassetid://8933346550",
        ["Gamepass"] = "rbxassetid://8933355899",
        ["Chain"] = "rbxassetid://81358145120405",
        ["Chain 2"] = "rbxassetid://132910145874066",
        ["Chain 3"] = "rbxassetid://128466395060514",
        ["Chain 4"] = "rbxassetid://73368670987191",
        ["Rope"] = "rbxassetid://78999022056924",
        ["Spring"] = "rbxassetid://18837732116",
        ["Circle"] = "rbxassetid://5367817750",
        ["Circle-Outline"] = "rbxassetid://12201347372",
        ["Triangle"] = "rbxassetid://4704920160",
        ["Triangle-Outline"] = "rbxassetid://94666748694025",
        ["Square"] = "rbxassetid://14657415531",
        ["Square-Outline"] = "rbxassetid://16540246889",
        ["Heart"] = "rbxassetid://89015294175898",
        ["Heart-Outline"] = "rbxassetid://125373934805238",
        ["Moon"] = "rbxassetid://9013498676",
        ["Dots"] = "rbxassetid://9169659357",
        ["Bubble"] = "rbxassetid://1249690853",
        ["Star"] = "rbxassetid://5639840603",
        ["Robux"] = "rbxassetid://11560341132",
        ["Roblox-Logo"] = "rbxassetid://12348119032",
        ["Brick"] = "rbxassetid://4430903072",
        ["Studs"] = "rbxassetid://15539356451",
        ["Fire"] = "rbxassetid://18654087326",
        ["Lazar"] = "rbxassetid://8922958725",
        ["Spider-Web"] = "rbxassetid://123815660139244",
        ["Smoke"] = "rbxassetid://12900071392",
        ["Audio-Visualiser"] = "rbxassetid://81588563590679",
        ["Pulse"] = "rbxassetid://82163767314193",
        ["Arrow"] = "rbxassetid://9006027964",
        ["Arrow 2"] = "rbxassetid://10249261576",
    }
    EchoLineTextureIds.__Values = {
        "Low Quality", "Non-Gamepass", "Gamepass", "Chain", "Chain 2", "Chain 3", "Chain 4",
        "Rope", "Spring", "Circle", "Circle-Outline", "Triangle", "Triangle-Outline", "Square",
        "Square-Outline", "Heart", "Heart-Outline", "Moon", "Dots", "Bubble", "Star", "Robux",
        "Roblox-Logo", "Brick", "Studs", "Fire", "Lazar", "Spider-Web", "Smoke", "Audio-Visualiser",
        "Pulse", "Arrow", "Arrow 2"
    }

    EchoLineTextureIds.__CleanId = function(text)
        return tostring(text or ""):gsub("%D", "")
    end

    EchoLineTextureIds.__ToAssetId = function(id)
        id = EchoLineTextureIds.__CleanId(id)
        if id == "" then return "" end
        return "rbxassetid://" .. id
    end

LineFns.lineNotify = function(textureName)
    _G.__EchoNotify("Line changed to " .. tostring(textureName), "line_changed_" .. tostring(textureName), 3, "LineChanged")
        local sound = Instance.new("Sound")
        sound.Name = "EchoLineChangeSound"
        sound.SoundId = LineState.lineNotificationSoundId
        sound.Volume = 1
        sound.RollOffMaxDistance = 10000
        sound.Parent = LineState.LineSoundService
        pcall(function() sound:Play() end)
        LineState.LineDebris:AddItem(sound, 3)
    end

    LineFns.lineCaptureOriginalSound = function(soundObj)
        if not soundObj or not soundObj:IsA("Sound") then return end
        if soundObj:GetAttribute("EchoOriginalSoundId") == nil then
            soundObj:SetAttribute("EchoOriginalSoundId", soundObj.SoundId or "")
        end
    end

    LineFns.lineSetSoundId = function(soundObj, newId)
        if not soundObj or not soundObj:IsA("Sound") or not newId or newId == "" then return end
        if soundObj.SoundId == newId then return end
        local wasPlaying = false
        pcall(function() wasPlaying = soundObj.IsPlaying end)
        pcall(function() soundObj:Stop() end)
        soundObj.SoundId = newId
        if wasPlaying then
            pcall(function()
                soundObj.TimePosition = 0
                soundObj:Play()
            end)
        end
    end

    LineFns.lineRestoreOriginalSound = function(soundObj)
        if not soundObj or not soundObj:IsA("Sound") then return end
        LineFns.lineCaptureOriginalSound(soundObj)
        local originalId = soundObj:GetAttribute("EchoOriginalSoundId")
        if originalId and originalId ~= "" and soundObj.SoundId ~= originalId then
            LineFns.lineSetSoundId(soundObj, originalId)
        end
    end

    LineFns.lineGrabPart = function()
        local gp = VisualWorkspace:FindFirstChild("GrabParts")
        return gp and gp:FindFirstChild("GrabPart")
    end

    LineFns.lineBeamPart = function()
        local gp = VisualWorkspace:FindFirstChild("GrabParts")
        return gp and gp:FindFirstChild("BeamPart")
    end

    LineFns.lineSetMode = function(mode)
        if LineState.lineToggleSyncing then return end
        LineState.lineToggleSyncing = true
        if mode == "selected" then
            LineState.lineUseSelectedTexture = true
            LineState.lineUseCustomTexture = false
            if Toggles.EchoLineUseSelectedTexture then Toggles.EchoLineUseSelectedTexture:SetValue(true) end
            if Toggles.EchoLineUseCustomTexture then Toggles.EchoLineUseCustomTexture:SetValue(false) end
        else
            LineState.lineUseSelectedTexture = false
            LineState.lineUseCustomTexture = true
            if Toggles.EchoLineUseSelectedTexture then Toggles.EchoLineUseSelectedTexture:SetValue(false) end
            if Toggles.EchoLineUseCustomTexture then Toggles.EchoLineUseCustomTexture:SetValue(true) end
        end
        LineState.lineToggleSyncing = false
    end

    LineFns.lineEnsureMode = function()
        local hasCustom = EchoLineTextureIds.__CleanId(LineState.lineCustomTextureId) ~= ""
        if LineState.lineUseCustomTexture and not hasCustom then
            LineState.lineUseCustomTexture = false
            LineState.lineUseSelectedTexture = true
        elseif LineState.lineUseSelectedTexture and LineState.lineUseCustomTexture then
            LineState.lineUseCustomTexture = false
        end
    end

    LineFns.lineActiveTexture = function()
        LineFns.lineEnsureMode()
        if LineState.lineUseCustomTexture then
            local custom = EchoLineTextureIds.__ToAssetId(LineState.lineCustomTextureId)
            if custom ~= "" then return custom, "Custom" end
        end
        if LineState.lineUseSelectedTexture then
            return EchoLineTextureIds[LineState.lineSelectedTexture] or "", LineState.lineSelectedTexture
        end
        return "", "Default"
    end

    LineFns.lineApplyBeam = function()
        local beamPart = LineFns.lineBeamPart()
        if not beamPart then return end
        local beam = beamPart:FindFirstChild("GrabBeam")
        if not beam or not beam:IsA("Beam") then return end
        local texture = LineFns.lineActiveTexture()
        beam.Texture = texture
        if LineState.lineSpeedEnabled then beam.TextureSpeed = LineState.lineTextureSpeed end
        if LineState.lineLengthEnabled then beam.TextureLength = LineState.lineTextureLength end
    end

    LineFns.lineApplySound = function()
        local gp = LineFns.lineGrabPart()
        if not gp then return end
        local sound = gp:FindFirstChild("AttachSound")
        if not sound or not sound:IsA("Sound") then return end
        LineFns.lineCaptureOriginalSound(sound)
        local custom = EchoLineTextureIds.__ToAssetId(LineState.lineCustomGrabSoundId)
        if custom ~= "" then LineFns.lineSetSoundId(sound, custom) else LineFns.lineRestoreOriginalSound(sound) end
    end

    LineFns.lineConnect = function()
        if LineState.lineHeartbeatConnection then return end
        LineState.lineHeartbeatConnection = LineState.LineRunService.Heartbeat:Connect(function(dt)
            LineFns.lineApplyBeam()
            LineState.lineSyncAccumulator = LineState.lineSyncAccumulator + dt
            if LineState.lineSyncAccumulator >= 0.15 then
                LineState.lineSyncAccumulator = 0
                LineFns.lineApplySound()
            end
        end)
    end

    local LineSelection = LineVisualGroup:AddDropdown("EchoLineTexture", {
        Values = EchoLineTextureIds.__Values,
        Default = 1,
        Text = "Select Line Texture",
        Tooltip = "Choose a grab beam texture",
        Callback = function(v)
            LineState.lineSelectedTexture = v
            LineFns.lineConnect()
            LineFns.lineApplyBeam()
            if LineState.lineUseSelectedTexture then LineFns.lineNotify(v) end
        end,
    })

    LineVisualGroup:AddInput("EchoLineCustomTexture", {
        Default = "",
        Numeric = false,
        Finished = true,
        Text = "Custom Texture",
        Placeholder = "Paste Texture ID...",
        Callback = function(text)
            LineState.lineCustomTextureId = EchoLineTextureIds.__CleanId(text)
            LineFns.lineConnect()
            if LineState.lineUseCustomTexture and LineState.lineCustomTextureId == "" then LineFns.lineSetMode("selected") end
            LineFns.lineApplyBeam()
            if LineState.lineUseCustomTexture and LineState.lineCustomTextureId ~= "" then LineFns.lineNotify("Custom") end
        end,
    })

    LineVisualGroup:AddToggle("EchoLineUseSelectedTexture", {
        Text = "Use Selected Texture",
        Default = true,
        Callback = function(v)
            if LineState.lineToggleSyncing then return end
            LineState.lineUseSelectedTexture = v and true or false
            if v then
                if LineState.lineUseCustomTexture then
                    LineState.lineToggleSyncing = true
                    LineState.lineUseCustomTexture = false
                    if Toggles.EchoLineUseCustomTexture then Toggles.EchoLineUseCustomTexture:SetValue(false) end
                    LineState.lineToggleSyncing = false
                end
                LineFns.lineConnect()
                LineFns.lineApplyBeam()
                LineFns.lineNotify(LineState.lineSelectedTexture)
            else
                LineFns.lineConnect()
                LineFns.lineApplyBeam()
            end
        end,
    })

    LineVisualGroup:AddToggle("EchoLineUseCustomTexture", {
        Text = "Use Custom Texture",
        Default = false,
        Callback = function(v)
            if LineState.lineToggleSyncing then return end
            if v and EchoLineTextureIds.__CleanId(LineState.lineCustomTextureId) == "" then
                LineState.lineToggleSyncing = true
                if Toggles.EchoLineUseCustomTexture then Toggles.EchoLineUseCustomTexture:SetValue(false) end
                LineState.lineUseCustomTexture = false
                LineState.lineToggleSyncing = false
                Library:Notify("Paste a custom texture ID first.")
                return
            end
            LineState.lineUseCustomTexture = v
            if v then
                LineFns.lineSetMode("custom")
                LineFns.lineConnect()
                LineFns.lineApplyBeam()
                LineFns.lineNotify("Custom")
            elseif not LineState.lineUseSelectedTexture then
                LineFns.lineSetMode("selected")
                LineFns.lineConnect()
                LineFns.lineApplyBeam()
            else
                LineFns.lineConnect()
                LineFns.lineApplyBeam()
            end
        end,
    })

    LineVisualGroup:AddDivider()
    LineVisualGroup:AddToggle("EchoLineSpeed", {
        Text = "Enable Custom Speed",
        Default = false,
        Callback = function(v)
            LineState.lineSpeedEnabled = v
            LineFns.lineConnect()
            LineFns.lineApplyBeam()
        end,
    })
    LineVisualGroup:AddSlider("EchoLineSpeedValue", {
        Text = "Grab Texture Speed",
        Default = 1,
        Min = 0,
        Max = 100,
        Rounding = 0,
        Callback = function(v)
            LineState.lineTextureSpeed = v
            if LineState.lineSpeedEnabled then
                LineFns.lineApplyBeam()
            end
        end,
    })
    LineVisualGroup:AddToggle("EchoLineLength", {
        Text = "Enable Custom Length",
        Default = false,
        Callback = function(v)
            LineState.lineLengthEnabled = v
            LineFns.lineConnect()
            LineFns.lineApplyBeam()
        end,
    })
    LineVisualGroup:AddSlider("EchoLineLengthValue", {
        Text = "Grab Texture Length",
        Default = 1,
        Min = 0,
        Max = 500,
        Rounding = 0,
        Callback = function(v)
            LineState.lineTextureLength = v
            if LineState.lineLengthEnabled then
                LineFns.lineApplyBeam()
            end
        end,
    })
    LineVisualGroup:AddInput("EchoLineGrabSound", {
        Default = "",
        Numeric = false,
        Finished = true,
        Text = "Custom Grab Sound ID",
        Placeholder = "Paste Sound ID...",
        Callback = function(text)
            LineState.lineCustomGrabSoundId = EchoLineTextureIds.__CleanId(text)
            LineFns.lineApplySound()
        end,
    })

    VisualWorkspace.DescendantAdded:Connect(function(d)
        if d:IsA("Beam") and d.Name == "GrabBeam" then
            task.defer(LineFns.lineApplyBeam)
        elseif d:IsA("Sound") and d.Name == "AttachSound" then
            LineFns.lineCaptureOriginalSound(d)
            task.defer(LineFns.lineApplySound)
        elseif d.Name == "GrabPart" or d.Name == "BeamPart" then
            task.defer(LineFns.lineApplyBeam)
            task.defer(LineFns.lineApplySound)
        end
    end)

    task.spawn(function()
        LineFns.lineConnect()
        LineFns.lineApplyBeam()
        LineFns.lineApplySound()
    end)
end

local ShaderGroup = VisualsTab:AddRightGroupbox("Realistic Tab", "palette")

local ShaderEnabled = false
local SelectedShader = "Morning"
local PShadeEffects = {}
local PShadeCreated = {}
local PShadeBackup = nil

-- PShade Ultimate presets.
local PShadePresets = {
    ["Morning"] = {ccB=-0.02,ccC=0.8,ccS=-0.5,ccT=Color3.fromRGB(100,150,200),bi=0.2,bs=5,bt=0.8,blur=2,amb=Color3.fromRGB(10,10,10),time=7.5,lat=44,bright=1.5,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(200,200,200),eds=0.1,ess=0.1,gs=true,oa=Color3.fromRGB(10,10,10),exp=0.3,cover=0.6,density=0.36,cloud=Color3.fromRGB(255,255,255),ad=0.2,ao=0.5,ac=Color3.fromRGB(70,120,170),adec=Color3.fromRGB(10,50,100),ag=0.3,ah=1,dof=0.5,focus=15,radius=5,near=0.5},
    ["Midday"] = {ccB=0.1,ccC=0.5,ccS=-0.3,ccT=Color3.fromRGB(242,243,243),bi=0.3,bs=10,bt=0.8,blur=5,amb=Color3.fromRGB(2,2,2),time=8,lat=-15.12,bright=3.25,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(255,247,237),eds=0.203,ess=0.255,gs=true,oa=Color3.fromRGB(51,54,67),exp=0.85,cover=0.75,density=0.26,cloud=Color3.fromRGB(255,255,255),ad=0.364,ao=0.556,ac=Color3.fromRGB(175,221,255),adec=Color3.fromRGB(13,105,172),ag=0.36,ah=0.72,dof=0.277,focus=21.54,radius=16.77,near=0.277},
    ["Afternoon"] = {ccB=0.1,ccC=0.5,ccS=-0.312,ccT=Color3.fromRGB(242,243,243),bi=0.31234,bs=10,bt=0.843,blur=5,amb=Color3.fromRGB(33,33,33),time=14,lat=-15.12,bright=2.25,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(255,247,237),eds=0.203,ess=0.255,gs=true,oa=Color3.fromRGB(51,54,67),exp=0.85,cover=0.45,density=0.23,cloud=Color3.fromRGB(255,255,255),ad=0.364,ao=0.556,ac=Color3.fromRGB(175,221,255),adec=Color3.fromRGB(13,105,172),ag=0.36,ah=0.72,dof=0.217,focus=21.54,radius=16.77,near=0.277},
    ["Evening"] = {ccB=0.1,ccC=0.5,ccS=-0.3,ccT=Color3.fromRGB(255,205,185),bi=0.3234,bs=10,bt=0.813,blur=5,amb=Color3.fromRGB(2,2,2),time=16,lat=45,bright=2.25,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(255,247,237),eds=0.203,ess=0.215,gs=true,oa=Color3.fromRGB(0,0,0),exp=0.65,cover=0.55,density=0.43,cloud=Color3.fromRGB(199,175,166),ad=0.364,ao=5.556,ac=Color3.fromRGB(199,175,166),adec=Color3.fromRGB(44,39,33),ag=0.36,ah=1.72,dof=0.217,focus=21.54,radius=16.77,near=0.277},
    ["Night"] = {ccB=-0.06,ccC=-0.02,ccS=-0.2,ccT=Color3.fromRGB(242,243,243),bi=0.34,bs=10,bt=0.813,blur=5,amb=Color3.fromRGB(33,33,33),time=20,lat=-15,bright=3.25,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(255,247,237),eds=0.203,ess=0.255,gs=true,oa=Color3.fromRGB(51,54,67),exp=0.85,cover=0.65,density=0.33,cloud=Color3.fromRGB(255,255,255),ad=0.264,ao=0.156,ac=Color3.fromRGB(175,221,255),adec=Color3.fromRGB(13,105,172),ag=0.36,ah=1.72,dof=0.217,focus=11.54,radius=16.77,near=0.277},
    ["Midnight"] = {ccB=-0.1,ccC=-0.05,ccS=-0.3,ccT=Color3.fromRGB(242,243,243),bi=0.1,bs=10,bt=0.81,blur=6,amb=Color3.fromRGB(33,33,33),time=0,lat=0,bright=2.25,csb=Color3.fromRGB(0,0,0),cst=Color3.fromRGB(255,247,237),eds=0.203,ess=0.255,gs=true,oa=Color3.fromRGB(51,54,67),exp=0.15,cover=0.75,density=0.33,cloud=Color3.fromRGB(255,255,255),ad=0.264,ao=0.156,ac=Color3.fromRGB(175,221,255),adec=Color3.fromRGB(13,105,172),ag=0.16,ah=0.72,dof=0.217,focus=11.54,radius=16.77,near=0.277},
}
local PShadeValues = {"Morning","Midday","Afternoon","Evening","Night","Midnight"}

local function savePShadeBackup()
    if PShadeBackup then return end
    PShadeBackup = {
        Lighting = {
            Ambient=Lighting.Ambient, ClockTime=Lighting.ClockTime, GeographicLatitude=Lighting.GeographicLatitude, Brightness=Lighting.Brightness,
            ColorShift_Bottom=Lighting.ColorShift_Bottom, ColorShift_Top=Lighting.ColorShift_Top, EnvironmentDiffuseScale=Lighting.EnvironmentDiffuseScale,
            EnvironmentSpecularScale=Lighting.EnvironmentSpecularScale, GlobalShadows=Lighting.GlobalShadows, OutdoorAmbient=Lighting.OutdoorAmbient,
            ExposureCompensation=Lighting.ExposureCompensation, FogEnd=Lighting.FogEnd, FogStart=Lighting.FogStart, FogColor=Lighting.FogColor,
        },
        Effects = {},
        Clouds = nil,
    }

    local function saveEffect(ClassName, Name, Properties)
        local obj = Lighting:FindFirstChild(Name)
        if obj and obj:IsA(ClassName) then
            local values = {}
            for _, property in ipairs(Properties) do
                local ok, value = pcall(function() return obj[property] end)
                if ok then values[property] = value end
            end
            PShadeBackup.Effects[obj] = {Properties=values, Created=false}
        end
    end

    saveEffect("ColorCorrectionEffect", "EchoPShadeColorCorrection", {"Enabled","Brightness","Contrast","Saturation","TintColor"})
    saveEffect("BloomEffect", "EchoPShadeBloom", {"Enabled","Intensity","Size","Threshold"})
    saveEffect("BlurEffect", "EchoPShadeBlur", {"Enabled","Size"})
    saveEffect("Atmosphere", "EchoPShadeAtmosphere", {"Enabled","Density","Offset","Color","Decay","Glare","Haze"})
    saveEffect("DepthOfFieldEffect", "EchoPShadeDepthOfField", {"Enabled","FarIntensity","FocusDistance","InFocusRadius","NearIntensity"})

    local terrain = Workspace:FindFirstChildOfClass("Terrain")
    if terrain then
        local clouds = terrain:FindFirstChild("EchoPShadeClouds")
        if clouds and clouds:IsA("Clouds") then
            PShadeBackup.Clouds = {Object=clouds, Properties={Cover=clouds.Cover,Density=clouds.Density,Color=clouds.Color,Enabled=clouds.Enabled}, Created=false}
        end
    end
end

local function getPShadeEffect(ClassName, Name, Parent)
    local Existing=Parent:FindFirstChild(Name)
    if Existing and Existing:IsA(ClassName) then
        if PShadeBackup and not PShadeBackup.Effects[Existing] then
            local properties = {}
            local list = {
                ColorCorrectionEffect={"Enabled","Brightness","Contrast","Saturation","TintColor"},
                BloomEffect={"Enabled","Intensity","Size","Threshold"},
                BlurEffect={"Enabled","Size"},
                Atmosphere={"Enabled","Density","Offset","Color","Decay","Glare","Haze"},
                DepthOfFieldEffect={"Enabled","FarIntensity","FocusDistance","InFocusRadius","NearIntensity"},
            }
            for _, property in ipairs(list[ClassName] or {}) do
                local ok, value = pcall(function() return Existing[property] end)
                if ok then properties[property] = value end
            end
            PShadeBackup.Effects[Existing] = {Properties=properties, Created=false}
        end
        return Existing
    end
    local Effect=Instance.new(ClassName)
    Effect.Name=Name
    Effect.Parent=Parent
    PShadeEffects[#PShadeEffects+1]=Effect
    PShadeCreated[Effect]=true
    if PShadeBackup then PShadeBackup.Effects[Effect] = {Properties={}, Created=true} end
    return Effect
end

local function resetPShade()
    for _,Effect in ipairs(PShadeEffects) do
        pcall(function()
            if Effect and Effect.Parent then Effect:Destroy() end
        end)
    end
    PShadeEffects={}
    PShadeCreated={}

    if PShadeBackup then
        local lightingBackup = PShadeBackup.Lighting or {}
        for Property,Value in pairs(lightingBackup) do
            pcall(function() Lighting[Property]=Value end)
        end
        for obj,data in pairs(PShadeBackup.Effects or {}) do
            pcall(function()
                if obj and obj.Parent and data and data.Properties then
                    for Property,Value in pairs(data.Properties) do obj[Property]=Value end
                end
            end)
        end
        if PShadeBackup.Clouds and PShadeBackup.Clouds.Object then
            local clouds = PShadeBackup.Clouds.Object
            pcall(function()
                if clouds.Parent then
                    for Property,Value in pairs(PShadeBackup.Clouds.Properties or {}) do clouds[Property]=Value end
                end
            end)
        end
    end
    PShadeBackup=nil
end

local function applyPShade(Name)
    local P=PShadePresets[Name]
    if not P then return false end
    savePShadeBackup()
    local CC=getPShadeEffect("ColorCorrectionEffect","EchoPShadeColorCorrection",Lighting)
    local Bloom=getPShadeEffect("BloomEffect","EchoPShadeBloom",Lighting)
    local Blur=getPShadeEffect("BlurEffect","EchoPShadeBlur",Lighting)
    local Atmos=getPShadeEffect("Atmosphere","EchoPShadeAtmosphere",Lighting)
    local DOF=getPShadeEffect("DepthOfFieldEffect","EchoPShadeDepthOfField",Lighting)
    local Clouds=getPShadeEffect("Clouds","EchoPShadeClouds",Workspace.Terrain)
    local L=Lighting
    pcall(function() L.Ambient=P.amb;L.ClockTime=P.time;L.GeographicLatitude=P.lat;L.Brightness=P.bright;L.ColorShift_Bottom=P.csb;L.ColorShift_Top=P.cst;L.EnvironmentDiffuseScale=P.eds;L.EnvironmentSpecularScale=P.ess;L.GlobalShadows=P.gs;L.OutdoorAmbient=P.oa;L.ExposureCompensation=P.exp;L.FogEnd=math.huge;L.FogStart=math.huge;L.FogColor=Color3.fromRGB(255,255,255) end)
    pcall(function() CC.Enabled=true;CC.Brightness=P.ccB;CC.Contrast=P.ccC;CC.Saturation=P.ccS;CC.TintColor=P.ccT end)
    pcall(function() Bloom.Enabled=true;Bloom.Intensity=P.bi;Bloom.Size=P.bs;Bloom.Threshold=P.bt end)
    pcall(function() Blur.Enabled=false;Blur.Size=P.blur end)
    pcall(function() Atmos.Enabled=true;Atmos.Density=P.ad;Atmos.Offset=P.ao;Atmos.Color=P.ac;Atmos.Decay=P.adec;Atmos.Glare=P.ag;Atmos.Haze=P.ah end)
    pcall(function() DOF.Enabled=true;DOF.FarIntensity=P.dof;DOF.FocusDistance=P.focus;DOF.InFocusRadius=P.radius;DOF.NearIntensity=P.near end)
    pcall(function() Clouds.Cover=P.cover;Clouds.Density=P.density;Clouds.Color=P.cloud end)
    return true
end

local function toggleShader(Value)
    ShaderEnabled=Value
    if Value then applyPShade(SelectedShader) else resetPShade() end
end

local function changeShader(Value)
    if PShadePresets[Value] then
        SelectedShader=Value
        if ShaderEnabled then applyPShade(Value) end
    end
end

ShaderGroup:AddToggle("ShaderToggle",{Text="Enable Shaders",Default=false,Callback=toggleShader})
ShaderGroup:AddDropdown("ShaderDropdown",{Text="Shader Selection",Values=PShadeValues,Default=SelectedShader,Callback=changeShader})
_G.__EchoWorldTimeValue=_G.__EchoWorldTimeValue or 12
ShaderGroup:AddSlider("EchoWorldTime",{Text="World Time (0-24)",Default=12,Min=0,Max=24,Rounding=1,Callback=function(Value)_G.__EchoWorldTimeValue=Value;pcall(function()Lighting.ClockTime=Value end)end})
ShaderGroup:AddButton({Text="Reset to Default",Func=function()ShaderEnabled=false;resetPShade()end})
ShaderGroup:AddButton({
    Text="Load Pshade",
    Func=function()
        local Ok,Source=pcall(function() return EchoHttpGet("https://raw.githubusercontent.com/randomstring0/pshade-ultimate/refs/heads/main/src/cd.lua") end)
        if not Ok or type(Source)~="string" then warn("[Echo] PShade Ultimate error:",tostring(Source));return end
        local CompileOk,Chunk=pcall(EchoLoadstring,Source)
        if not CompileOk or type(Chunk)~="function" then warn("[Echo] PShade Ultimate compile error:",tostring(Chunk));return end
        local RunOk,Err=pcall(Chunk)
        if not RunOk then warn("[Echo] PShade Ultimate error:",tostring(Err)) end
    end,
})

if Library and Library.OnUnload then
    Library:OnUnload(function()
        pcall(resetPShade)
    end)
end

-- ============================================================
-- ANIME VISUALS
-- ============================================================
_EchoAnimeBox = VisualsTab:AddRightTabbox("Anime Visuals")
_G.__EchoAnimeTab = _EchoAnimeBox:AddTab("Anime Visuals", "image")

_G.__EchoAnimeVisualState = _G.__EchoAnimeVisualState or {
    Gui = nil,
    Image1On = false,
    Image2On = false,
    PhysicsConnection = nil,

    HamakazeSize = 100,
    FernSize = 100,
}

-- ============================================================
-- IMAGE DATA
-- ============================================================
_G.__EchoAnimeImageURL =
    "https://p7.hiclipart.com/preview/604/559/584/kantai-collection-japanese-destroyer-hamakaze-anime-hair-japanese-destroyer-shimakaze-a-girl-with-a-ponytail-thumbnail.jpg"

_G.__EchoAnimeImageFile =
    "Echo_AnimeVisual_Hamakaze_Compressed_v4.png"

_G.__EchoAnimeImageData =
    _G.__EchoAnimeImageData or [[
iVBORw0KGgoAAAANSUhEUgAAATEAAAHRCAMAAAA1/37Y
]]

_G.__EchoAnimeImage2URL =
    "https://files.catbox.moe/gjwtnu.png"

_G.__EchoAnimeImage2File =
    "Echo_AnimeVisual_Fern_Compressed_v1.png"

_G.__EchoAnimeImage2Data =
    _G.__EchoAnimeImage2Data or [[]]

_G.__EchoAnimeLoadImage =
    EchoLoadAnimeImage

-- ============================================================
-- FERN LOADER
-- ============================================================
local function EchoLoadAnimeImage2()

    if _G.__EchoAnimeImage2Data
        and _G.__EchoAnimeImage2Data ~= "" then

        return EchoLoadAnimeImage(
            _G.__EchoAnimeImage2File,
            _G.__EchoAnimeImage2Data
        )
    end

    local url =
        _G.__EchoAnimeImage2URL

    if not url
        or url == "" then

        return nil
    end

    local getter =
        getcustomasset
        or getsynasset

    if type(getter) ~= "function" then
        return nil
    end

    if type(isfile) == "function"
        and isfile(_G.__EchoAnimeImage2File) then

        local ok, asset =
            pcall(
                getter,
                _G.__EchoAnimeImage2File
            )

        if ok
            and type(asset) == "string"
            and asset ~= "" then

            return asset
        end
    end

    if type(writefile) ~= "function" then
        return nil
    end

    local ok, body =
        pcall(function()
            return game:HttpGet(url)
        end)

    if not ok
        or type(body) ~= "string"
        or #body < 100 then

        return nil
    end

    local wrote =
        pcall(
            writefile,
            _G.__EchoAnimeImage2File,
            body
        )

    if not wrote then
        return nil
    end

    for _ = 1, 20 do

        local ok2, asset =
            pcall(
                getter,
                _G.__EchoAnimeImage2File
            )

        if ok2
            and type(asset) == "string"
            and asset ~= "" then

            return asset
        end

        task.wait(0.05)
    end

    return nil
end

-- ============================================================
-- DESTROY
-- ============================================================
function _G.__EchoDestroyAnimeVisuals()

    local S =
        _G.__EchoAnimeVisualState

    if S.PhysicsConnection then

        pcall(function()
            S.PhysicsConnection:Disconnect()
        end)

        S.PhysicsConnection = nil
    end

    if S.Gui then

        pcall(function()

            for _, name in ipairs({
                "Hamakaze",
                "Fern",
            }) do

                local image =
                    S.Gui:FindFirstChild(
                        name,
                        true
                    )

                if image
                    and image:IsA("ImageLabel") then

                    image.Image = ""
                end
            end

            S.Gui:Destroy()

        end)

        S.Gui = nil
    end

    local parent =
        (_G.__EchoGetGuiParent
            and _G.__EchoGetGuiParent())
        or game:GetService("CoreGui")

    pcall(function()

        local oldGui =
            parent:FindFirstChild(
                "EchoAnimeVisuals"
            )

        if oldGui then
            oldGui:Destroy()
        end
    end)
end

-- ============================================================
-- DRAGGING
-- ============================================================
local function EchoMakeAnimeDraggable(image)

    if not image then
        return
    end

    local UserInputService =
        game:GetService("UserInputService")

    local dragging = false
    local dragInput = nil
    local dragStart = nil
    local startPosition = nil

    image.Active = true

    local function update(input)

        if not dragging then
            return
        end

        local delta =
            input.Position - dragStart

        image.Position =
            UDim2.new(
                startPosition.X.Scale,
                startPosition.X.Offset + delta.X,
                startPosition.Y.Scale,
                startPosition.Y.Offset + delta.Y
            )
    end

    image.InputBegan:Connect(function(input)

        if input.UserInputType
            == Enum.UserInputType.MouseButton1
            or input.UserInputType
            == Enum.UserInputType.Touch then

            dragging = true
            dragStart = input.Position
            startPosition = image.Position

            input.Changed:Connect(function()

                if input.UserInputState
                    == Enum.UserInputState.End then

                    dragging = false
                end
            end)
        end
    end)

    image.InputChanged:Connect(function(input)

        if input.UserInputType
            == Enum.UserInputType.MouseMovement
            or input.UserInputType
            == Enum.UserInputType.Touch then

            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)

        if input == dragInput then
            update(input)
        end
    end)
end

-- ============================================================
-- SHOW
-- ============================================================
function _G.__EchoShowAnimeVisuals()

    _G.__EchoDestroyAnimeVisuals()

    local S =
        _G.__EchoAnimeVisualState

    if not S.Image1On
        and not S.Image2On then

        return
    end

    local asset1
    local asset2

    if S.Image1On then

        asset1 =
            _G.__EchoAnimeLoadImage(
                _G.__EchoAnimeImageFile,
                _G.__EchoAnimeImageData
            )
    end

    if S.Image2On then
        asset2 =
            EchoLoadAnimeImage2()
    end

    if not asset1
        and not asset2 then

        pcall(function()

            Library:Notify({
                Description =
                    "Anime image could not be loaded.",
                Time = 4,
            })
        end)

        return
    end

    local gui =
        Instance.new("ScreenGui")

    gui.Name =
        "EchoAnimeVisuals"

    gui.ResetOnSpawn =
        false

    gui.IgnoreGuiInset =
        true

    gui.ZIndexBehavior =
        Enum.ZIndexBehavior.Sibling

    local parent =
        (_G.__EchoGetGuiParent
            and _G.__EchoGetGuiParent())
        or game:GetService("CoreGui")

    gui.Parent = parent

    -- ========================================================
    -- HAMAKAZE
    -- ========================================================
    if asset1 then

        local image =
            Instance.new("ImageLabel")

        image.Name =
            "Hamakaze"

        image.BackgroundTransparency =
            1

        image.BorderSizePixel =
            0

        image.Image =
            asset1

        image.ImageTransparency =
            0

        image.ScaleType =
            Enum.ScaleType.Fit

        image.AnchorPoint =
            Vector2.new(1, 0.5)

        image.Position =
            UDim2.new(
                1,
                -20,
                0.5,
                0
            )

        image.Size =
            UDim2.new(
                0,
                280,
                0,
                576
            )

        image.Rotation =
            0

        image.ZIndex =
            10

        image.Parent =
            gui

        EchoMakeAnimeDraggable(image)
    end

    -- ========================================================
    -- FERN
    -- ========================================================
    if asset2 then

        local image =
            Instance.new("ImageLabel")

        image.Name =
            "Fern"

        image.BackgroundTransparency =
            1

        image.BorderSizePixel =
            0

        image.Image =
            asset2

        image.ImageTransparency =
            0

        image.ScaleType =
            Enum.ScaleType.Fit

        image.AnchorPoint =
            Vector2.new(0, 0.5)

        image.Position =
            UDim2.new(
                0,
                20,
                0.5,
                0
            )

        image.Size =
            UDim2.new(
                0,
                280,
                0,
                576
            )

        image.Rotation =
            0

        image.ZIndex =
            10

        image.Parent =
            gui

        EchoMakeAnimeDraggable(image)
    end

    S.Gui =
        gui

    -- ========================================================
    -- CURSOR PHYSICS
    -- ========================================================
    local UIS =
        game:GetService("UserInputService")

    local RunService =
        game:GetService("RunService")

    local ImagePhysics = {}

    local function setupImage(
        image,
        imageType
    )

        if not image then
            return nil
        end

        return {
            Image = image,

            ImageType =
                imageType,

            BaseSize =
                image.Size,

            Rotation =
                0,

            RotationVelocity =
                0,

            ScaleX =
                1,

            ScaleY =
                1,

            ScaleVelocityX =
                0,

            ScaleVelocityY =
                0,

            LastCursorX =
                nil,

            LastCursorY =
                nil,

            CursorVelocityX =
                0,

            CursorVelocityY =
                0,

            Time =
                math.random() * 10,
        }
    end

    ImagePhysics.Hamakaze =
        setupImage(
            gui:FindFirstChild("Hamakaze"),
            "Hamakaze"
        )

    ImagePhysics.Fern =
        setupImage(
            gui:FindFirstChild("Fern"),
            "Fern"
        )

    local DetectionRadius =
        450

    local MaxRotation =
        4

    local MaxStretch =
        0.035

    local Spring =
        22

    local Damping =
        8

    local function updateImage(
        P,
        dt
    )

        local image =
            P and P.Image

        if not image
            or not image.Parent then

            return
        end

        local mouse =
            UIS:GetMouseLocation()

        local position =
            image.AbsolutePosition

        local size =
            image.AbsoluteSize

        local centerX =
            position.X
            + size.X * 0.5

        local centerY =
            position.Y
            + size.Y * 0.5

        local deltaX =
            mouse.X - centerX

        local deltaY =
            mouse.Y - centerY

        local distance =
            math.sqrt(
                deltaX * deltaX
                + deltaY * deltaY
            )

        if P.LastCursorX then

            P.CursorVelocityX =
                (
                    mouse.X
                    - P.LastCursorX
                )
                / math.max(
                    dt,
                    0.001
                )

            P.CursorVelocityY =
                (
                    mouse.Y
                    - P.LastCursorY
                )
                / math.max(
                    dt,
                    0.001
                )
        end

        P.LastCursorX =
            mouse.X

        P.LastCursorY =
            mouse.Y

        P.CursorVelocityX =
            math.clamp(
                P.CursorVelocityX,
                -2500,
                2500
            )

        P.CursorVelocityY =
            math.clamp(
                P.CursorVelocityY,
                -2500,
                2500
            )

        local proximity =
            0

        if distance < DetectionRadius then

            proximity =
                1
                - distance / DetectionRadius

            proximity =
                math.clamp(
                    proximity,
                    0,
                    1
                )

            proximity =
                proximity
                * proximity
                * (
                    3
                    - 2 * proximity
                )
        end

        local cursorSpeed =
            math.sqrt(
                P.CursorVelocityX
                    * P.CursorVelocityX
                +
                P.CursorVelocityY
                    * P.CursorVelocityY
            )

        cursorSpeed =
            math.clamp(
                cursorSpeed / 1600,
                0,
                1
            )

        local directionX =
            0

        if distance > 0.01 then

            directionX =
                deltaX / distance
        end

        local movementRotation =
            math.clamp(
                P.CursorVelocityX
                * 0.0025,
                -1.5,
                1.5
            )

        local targetRotation =
            (
                directionX
                * proximity
                * MaxRotation
                * 0.65
            )
            +
            movementRotation
            * proximity

        targetRotation =
            math.clamp(
                targetRotation,
                -MaxRotation,
                MaxRotation
            )

        local rotationForce =
            (
                targetRotation
                - P.Rotation
            )
            * Spring

        P.RotationVelocity =
            P.RotationVelocity
            + rotationForce * dt

        P.RotationVelocity =
            P.RotationVelocity
            * math.exp(
                -Damping * dt
            )

        P.Rotation =
            P.Rotation
            + P.RotationVelocity * dt

        P.Time =
            P.Time
            + dt
            * (
                5
                + proximity * 16
                + cursorSpeed * 8
            )

        local wave =
            math.sin(P.Time)

        local wave2 =
            math.sin(
                P.Time * 1.7
            )

        P.Rotation =
            P.Rotation
            + (
                wave
                * proximity
                * cursorSpeed
                * 1.2
                * dt
                * 8
            )

        local targetScaleX =
            1
            + (
                wave2
                * cursorSpeed
                * proximity
                * MaxStretch
            )

        local targetScaleY =
            1
            - (
                wave2
                * cursorSpeed
                * proximity
                * MaxStretch
            )

        local scaleForceX =
            (
                targetScaleX
                - P.ScaleX
            )
            * Spring

        P.ScaleVelocityX =
            P.ScaleVelocityX
            + scaleForceX * dt

        P.ScaleVelocityX =
            P.ScaleVelocityX
            * math.exp(
                -Damping * dt
            )

        P.ScaleX =
            P.ScaleX
            + P.ScaleVelocityX * dt

        local scaleForceY =
            (
                targetScaleY
                - P.ScaleY
            )
            * Spring

        P.ScaleVelocityY =
            P.ScaleVelocityY
            + scaleForceY * dt

        P.ScaleVelocityY =
            P.ScaleVelocityY
            * math.exp(
                -Damping * dt
            )

        P.ScaleY =
            P.ScaleY
            + P.ScaleVelocityY * dt

        -- ====================================================
        -- INDIVIDUAL IMAGE SIZE
        -- ====================================================
        local userScale = 1

        if P.ImageType == "Hamakaze" then

            userScale =
                (
                    tonumber(
                        S.HamakazeSize
                    )
                    or 100
                )
                / 100

        elseif P.ImageType == "Fern" then

            userScale =
                (
                    tonumber(
                        S.FernSize
                    )
                    or 100
                )
                / 100
        end

        local finalScaleX =
            userScale
            * P.ScaleX

        local finalScaleY =
            userScale
            * P.ScaleY

        image.Rotation =
            P.Rotation

        image.Size =
            UDim2.new(
                P.BaseSize.X.Scale
                    * finalScaleX,

                P.BaseSize.X.Offset
                    * finalScaleX,

                P.BaseSize.Y.Scale
                    * finalScaleY,

                P.BaseSize.Y.Offset
                    * finalScaleY
            )
    end

    S.PhysicsConnection =
        RunService.RenderStepped:Connect(
            function(dt)

                dt =
                    math.clamp(
                        dt,
                        0,
                        1 / 30
                    )

                updateImage(
                    ImagePhysics.Hamakaze,
                    dt
                )

                updateImage(
                    ImagePhysics.Fern,
                    dt
                )

                if not S.Gui
                    or not S.Gui.Parent then

                    if S.PhysicsConnection then

                        pcall(function()
                            S.PhysicsConnection:Disconnect()
                        end)

                        S.PhysicsConnection =
                            nil
                    end
                end
            end
        )
end

-- ============================================================
-- HAMAKAZE TOGGLE
-- ============================================================
_G.__EchoAnimeTab:AddToggle(
    "EchoAnimeImage1On",
    {
        Text = "Hamakaze",
        Default = false,

        Callback = function(v)

            _G.__EchoAnimeVisualState.Image1On =
                v and true or false

            _G.__EchoShowAnimeVisuals()
        end,
    }
)

-- ============================================================
-- HAMAKAZE SIZE
-- ============================================================
_G.__EchoAnimeTab:AddSlider(
    "EchoAnimeHamakazeSize",
    {
        Text = "Hamakaze Size",

        Default = 100,

        Min = 25,

        Max = 200,

        Rounding = 0,

        Suffix = "%",

        Callback = function(value)

            _G.__EchoAnimeVisualState.HamakazeSize =
                tonumber(value)
                or 100
        end,
    }
)

-- ============================================================
-- FERN TOGGLE
-- ============================================================
_G.__EchoAnimeTab:AddToggle(
    "EchoAnimeImage2On",
    {
        Text = "Fern",
        Default = false,

        Callback = function(v)

            _G.__EchoAnimeVisualState.Image2On =
                v and true or false

            _G.__EchoShowAnimeVisuals()
        end,
    }
)

-- ============================================================
-- FERN SIZE
-- ============================================================
_G.__EchoAnimeTab:AddSlider(
    "EchoAnimeFernSize",
    {
        Text = "Fern Size",

        Default = 100,

        Min = 25,

        Max = 200,

        Rounding = 0,

        Suffix = "%",

        Callback = function(value)

            _G.__EchoAnimeVisualState.FernSize =
                tonumber(value)
                or 100
        end,
    }
)

-- ============================================================
-- CLEANUP
-- ============================================================
pcall(function()

    Library:OnUnload(function()

        _G.__EchoDestroyAnimeVisuals()

    end)
end)

-- ============================================================
-- NOTIFICATIONS TAB (v13 — guaranteed balanced)
-- ============================================================
do
    local NotificationsTab = VisualTabBox:AddTab("Notifications", "bell")
    _G.__EchoNotificationsTab = NotificationsTab

    if _G.__EchoNotificationConnections then
        for _, c in pairs(_G.__EchoNotificationConnections) do
            pcall(function() c:Disconnect() end)
        end
    end
    _G.__EchoNotificationConnections = {}

    if _G.__EchoNotificationSoundFolder and _G.__EchoNotificationSoundFolder.Parent then
        pcall(function() _G.__EchoNotificationSoundFolder:Destroy() end)
    end
    _G.__EchoNotificationSoundFolder = nil

    if _G.__EchoNotificationsTabCleanup then
        pcall(_G.__EchoNotificationsTabCleanup)
        _G.__EchoNotificationsTabCleanup = nil
    end

    _G.__EchoNotificationState = {
        JoinNotify       = true,
        LeaveNotify      = true,
        RejoinNotify     = true,
        KickNotify       = true,
        TargetJoinNotify = true,
        PacketNotify     = false,
        BlackholeNotify  = true,
        FriendJoinNotify = false,
    }

    local EchoNotificationSounds = {
        ["Resonance Notification"] = "rbxassetid://97643101798871",
        ["GtaV"]                   = "rbxassetid://116627196004523",
        ["Steam"]                  = "rbxassetid://139308638407157",
        ["Popup"]                  = "rbxassetid://71450094482101",
        ["Ping"]                   = "rbxassetid://82418169358213",
        ["Correct"]                = "rbxassetid://110305285829402",
        ["CSGO - Headshot"]        = "rbxassetid://118849595584555",
    }

    local EchoNotificationSoundNames = {
        "Resonance Notification",
        "GtaV",
        "Steam",
        "Popup",
        "Ping",
        "Correct",
        "CSGO - Headshot",
    }

    local EchoNotificationEventSlots = {
        { key = "Startup",          label = "Startup",              default = "Popup"                },
        { key = "JoinNotify",       label = "Join Notification",    default = "Correct"              },
        { key = "LeaveNotify",      label = "Leave Notification",   default = "Ping"                 },
        { key = "RejoinNotify",     label = "Rejoin Notification",  default = "Correct"              },
        { key = "FriendJoinNotify", label = "Friend Join",          default = "Correct"              },
        { key = "TargetJoinNotify", label = "Target Join",          default = "Steam"                },
        { key = "KickNotify",       label = "Kick Notify",          default = "CSGO - Headshot"      },
        { key = "BlackholeNotify",  label = "Blackhole Notify",     default = "CSGO - Headshot"      },
        { key = "PacketNotify",     label = "Packet Lag Notify",    default = "Popup"                },
        { key = "AntiKickRemoved",  label = "Anti Kick Removed",    default = "Ping"                 },
        { key = "GucciActivated",   label = "Gucci Activated",      default = "Resonance Notification" },
        { key = "GucciDestroyed",   label = "Gucci Destroyed",      default = "Ping"                 },
        { key = "InventoryCleared", label = "Inventory Cleared",    default = "Resonance Notification" },
        { key = "AntiCheatWarn",    label = "Anti Cheat Warning",   default = "Resonance Notification" },
        { key = "LineChanged",      label = "Line Texture Changed", default = "Resonance Notification" },
        { key = "Default",          label = "Default / Fallback",   default = "Resonance Notification" },
    }

    local EchoEventDefaultSound = {}
    for _, slot in ipairs(EchoNotificationEventSlots) do
        EchoEventDefaultSound[slot.key] = slot.default
    end

    local EchoSoundToEventGuess = {
        ["Correct"]                = "JoinNotify",
        ["Ping"]                   = "LeaveNotify",
        ["GtaV"]                   = "KickNotify",
        ["Steam"]                  = "TargetJoinNotify",
        ["Popup"]                  = "PacketNotify",
        ["CSGO - Headshot"]        = "KickNotify",
        ["Resonance Notification"] = "Default",
    }

    _G.__EchoNotificationConfig = _G.__EchoNotificationConfig or {}
    local config = _G.__EchoNotificationConfig

    config.DefaultTime   = config.DefaultTime   or 4
    config.Volume        = config.Volume        or 1
    config.SoundName     = config.SoundName     or "Resonance Notification"
    config.SoundId       = EchoNotificationSounds[config.SoundName] or EchoNotificationSounds["Resonance Notification"]
    config.CustomSoundId = config.CustomSoundId or ""
    config.Place         = config.Place         or "Right"

    if config.Mode == nil or config.Mode == "" or config.Mode == "Simple" or config.Mode == "Silent" or config.Mode == "Per-Event" then
        config.Mode = "Default"
    end
    if config.Mode ~= "Default" and config.Mode ~= "Customize" then
        config.Mode = "Default"
    end

    if type(config.PerEvent) ~= "table" then
        config.PerEvent = {}
        for _, slot in ipairs(EchoNotificationEventSlots) do
            config.PerEvent[slot.key] = slot.default
        end
    else
        for _, slot in ipairs(EchoNotificationEventSlots) do
            if not config.PerEvent[slot.key] then
                config.PerEvent[slot.key] = slot.default
            end
        end
    end

    local EchoSoundFolder = Instance.new("Folder")
    EchoSoundFolder.Name = "EchoNotificationSounds"
    EchoSoundFolder.Parent = SoundService
    _G.__EchoNotificationSoundFolder = EchoSoundFolder

    local function resolveSoundId(requested)
        local mode = config.Mode or "Default"

        if mode == "Default" then
            if type(config.CustomSoundId) == "string" and config.CustomSoundId ~= "" then
                return config.CustomSoundId, "Custom"
            end
            local soundName = config.SoundName or "Resonance Notification"
            local soundId = EchoNotificationSounds[soundName] or config.SoundId or EchoNotificationSounds["Resonance Notification"]
            return soundId, soundName, "Global"
        end

        local eventKey = requested
        if not EchoEventDefaultSound[eventKey] then
            eventKey = EchoSoundToEventGuess[requested] or "Default"
        end

        local soundName = config.PerEvent[eventKey] or config.PerEvent.Default or config.SoundName or "Resonance Notification"
        local soundId = EchoNotificationSounds[soundName] or EchoNotificationSounds["Resonance Notification"]
        return soundId, soundName, eventKey
    end

    function _G.__EchoPlayNotificationSound(requested)
        local soundId, soundName = resolveSoundId(requested)
        if not soundId or soundId == "" then return false end

        return pcall(function()
            local sound = Instance.new("Sound")
            sound.Name = "EchoNotify_" .. tostring(soundName or "Default")
            sound.SoundId = soundId
            sound.Volume = math.clamp(tonumber(config.Volume) or 1, 0, 10)
            sound.PlayOnRemove = false
            sound.Parent = EchoSoundFolder

            if not sound.IsLoaded then
                local deadline = tick() + 2
                while not sound.IsLoaded and tick() < deadline do
                    task.wait(0.02)
                end
            end

            sound:Play()

            task.delay(6, function()
                if sound and sound.Parent then
                    pcall(function() sound:Destroy() end)
                end
            end)
        end)
    end

    _G.__EchoNotifyLastGlobal   = 0
    _G.__EchoNotifyKeyCooldowns = _G.__EchoNotifyKeyCooldowns or {}
    _G.__EchoNotifyDedupeCache  = _G.__EchoNotifyDedupeCache  or {}
    _G.__EchoLastPacketNotify   = 0
    _G.__EchoBlackholeWatch     = _G.__EchoBlackholeWatch     or {}
    _G.__EchoRejoinMemory       = _G.__EchoRejoinMemory       or {}

    local function hashOf(desc) return tostring(desc or "") end

    local function canEmit(key, hash)
        local now = tick()
        if now - (_G.__EchoNotifyDedupeCache[hash] or 0) < 3 then return false end
        if now - (_G.__EchoNotifyKeyCooldowns[key] or 0) < 4 then return false end
        return true
    end

    function _G.__EchoNotify(desc, key, duration, soundKey)
        local hash = hashOf(desc)
        key = key or hash
        if not canEmit(key, hash) then return false end

        local now = tick()
        _G.__EchoNotifyLastGlobal = now
        _G.__EchoNotifyKeyCooldowns[key] = now
        _G.__EchoNotifyDedupeCache[hash]  = now

        pcall(function()
            Library:Notify({
                Title        = "Echo",
                Description  = tostring(desc),
                Time         = duration or config.DefaultTime,
                Image        = "rbxassetid://" .. tostring(Logo),
                EchoSoundKey = soundKey or "__selected",
            })
        end)

        return true
    end

    function _G.__EchoNotifyTest(desc)
        pcall(function()
            Library:Notify({
                Title        = "Echo",
                Description  = tostring(desc or "Test notification"),
                Time         = 3,
                Image        = "rbxassetid://" .. tostring(Logo),
                EchoSoundKey = "__selected",
            })
        end)
        return true
    end

    if not Library.__EchoNotifyWrapped then
        Library.__EchoNotifyWrapped = true
        local baseNotify = Library.Notify
        Library.Notify = function(self, ...)
            local args = {...}
            local soundKey
            if type(args[1]) == "table" then
                if args[1].Title == nil or args[1].Title == "" then args[1].Title = "Echo" end
                if args[1].Image == nil then args[1].Image = "rbxassetid://" .. tostring(Logo) end
                soundKey = args[1].EchoSoundKey
                args[1].EchoSoundKey = nil
            elseif type(args[1]) == "string" then
                args[1] = { Title = "Echo", Description = args[1], Time = 3, Image = "rbxassetid://" .. tostring(Logo) }
                soundKey = "__selected"
            end
            task.defer(function() _G.__EchoPlayNotificationSound(soundKey) end)
            return baseNotify(self, table.unpack(args))
        end
    end

    function _G.__EchoApplyNotificationPlacement()
        local place = config.Place or "Right"
        local notifications = Library.Notifications
        local area

        if type(notifications) == "table" then
            for fakeBackground in pairs(notifications) do
                if fakeBackground and fakeBackground.Parent then
                    area = fakeBackground.Parent
                    break
                end
            end
        end
        if not area then return end

        if place == "Left" or place == "Right" then
            pcall(function() Library:SetNotifySide(place) end)
            return
        end
    end

    local connections = _G.__EchoNotificationConnections

    local function disconnectKey(name)
        local c = connections[name]
        if c then
            pcall(function() c:Disconnect() end)
            connections[name] = nil
        end
    end

    function _G.__EchoNotificationRefreshPlayerAdded()
        disconnectKey("PlayerAdded")
        local state = _G.__EchoNotificationState
        if not (state.JoinNotify or state.RejoinNotify or state.FriendJoinNotify or state.TargetJoinNotify) then return end

        connections.PlayerAdded = Players.PlayerAdded:Connect(function(player)
            task.spawn(function()
                if state.JoinNotify then
                    _G.__EchoNotify(player.DisplayName .. "(" .. player.Name .. ") Has Joined The Server", "join_" .. player.UserId, 6, "JoinNotify")
                end
                if state.RejoinNotify then
                    local rejoin = _G.__EchoRejoinMemory[player.UserId]
                    if rejoin then
                        _G.__EchoNotify(player.DisplayName .. "(" .. player.Name .. ") Has Rejoined", "rejoin_" .. player.UserId, 6, "RejoinNotify")
                        _G.__EchoRejoinMemory[player.UserId] = nil
                    end
                end
            end)
        end)
    end

    function _G.__EchoNotificationRefreshPlayerRemoving()
        disconnectKey("PlayerRemoving")
        local state = _G.__EchoNotificationState
        if not (state.LeaveNotify or state.RejoinNotify) then return end

        connections.PlayerRemoving = Players.PlayerRemoving:Connect(function(player)
            if state.RejoinNotify then
                _G.__EchoRejoinMemory[player.UserId] = { Time = tick(), Name = player.Name, DisplayName = player.DisplayName }
            end
            if state.LeaveNotify then
                _G.__EchoNotify(player.DisplayName .. "(" .. player.Name .. ") Has Left The Server", "leave_" .. player.UserId, 4, "LeaveNotify")
            end
        end)
    end

    function _G.__EchoNotificationRefreshPacket()
        disconnectKey("Packet")
        if not _G.__EchoNotificationState.PacketNotify then return end

        local events = ReplicatedStorage:FindFirstChild("GrabEvents")
        local extendGrabLine = events and events:FindFirstChild("ExtendGrabLine")
        if not extendGrabLine then return end

        connections.Packet = extendGrabLine.OnClientEvent:Connect(function(player, args)
            if typeof(args) ~= "string" or #args <= 300 then return end
            local now = tick()
            if now - _G.__EchoLastPacketNotify < 4 then return end
            _G.__EchoLastPacketNotify = now
            local sizeMB = math.round((#args / (1024 * 1024)) * 1000) / 1000
            local name = player and player.Name or "Unknown"
            _G.__EchoNotify(name .. " sent a large grab-line packet (" .. tostring(sizeMB) .. " MB).", "packet_" .. name, 5, "PacketNotify")
        end)
    end

    function _G.__EchoNotificationRefreshBlackhole()
        disconnectKey("BlackholeAdded")
        disconnectKey("BlackholeRemoving")
    end

    NotificationsTab:AddDropdown("EchoNotificationMode", {
        Values     = { "Default", "Customize" },
        Default    = config.Mode,
        Text       = "Sound Mode",
        Searchable = false,
        Multi      = false,
        Tooltip    = "Default: one sound for everything.\nCustomize: per-event assignments + presets.",
        Callback   = function(Value)
            local selected = Value
            if type(selected) == "table" then
                selected = selected[1]
                if not selected then
                    for key, enabled in pairs(Value) do
                        if enabled then
                            selected = key
                            break
                        end
                    end
                end
            end

            selected = tostring(selected or "Default")
            if selected ~= "Customize" then
                selected = "Default"
            end

            config.Mode = selected

            if selected == "Default" then
                config.SoundName = config.SoundName or "Resonance Notification"
                config.SoundId = EchoNotificationSounds[config.SoundName]
                    or EchoNotificationSounds["Resonance Notification"]
            end

            if _G.__EchoNotificationSyncMode then
                _G.__EchoNotificationSyncMode()
            end
        end,
    })

    local defaultSection = {}
    table.insert(defaultSection, NotificationsTab:AddLabel("<b>Default Mode</b>"))
    table.insert(defaultSection, NotificationsTab:AddDivider())

    table.insert(defaultSection, NotificationsTab:AddDropdown("EchoNotificationSide", {
        Values = { "Left", "Right" }, Default = config.Place, Text = "Notification Side",
        Searchable = false, Multi = false,
        Callback = function(Value)
            config.Place = Value
            pcall(function() Library:SetNotifySide(Value) end)
            _G.__EchoApplyNotificationPlacement()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddDropdown("EchoNotificationSound", {
        Values = EchoNotificationSoundNames, Default = config.SoundName, Text = "Notification Sound",
        Searchable = false, Multi = false,
        Callback = function(Value)
            config.SoundName = Value
            local id = EchoNotificationSounds[Value]
            if id then config.SoundId = id end
            _G.__EchoPlayNotificationSound()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddInput("EchoCustomNotificationSoundId", {
        Default = "", Numeric = false, Finished = true,
        Text = "Custom Notification Sound ID", Placeholder = "Paste Roblox Sound ID...",
        Callback = function(Text)
            local cleaned = tostring(Text or ""):gsub("%D", "")
            config.CustomSoundId = cleaned == "" and "" or ("rbxassetid://" .. cleaned)
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddButton({
        Text = "Test Notify",
        Func = function() _G.__EchoNotifyTest() end,
    }))

    table.insert(defaultSection, NotificationsTab:AddDivider())

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyJoin", {
        Text = "Join Notification", Default = true,
        Callback = function(v)
            _G.__EchoNotificationState.JoinNotify = v and true or false
            _G.__EchoNotificationRefreshPlayerAdded()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyLeave", {
        Text = "Leave Notification", Default = true,
        Callback = function(v)
            _G.__EchoNotificationState.LeaveNotify = v and true or false
            _G.__EchoNotificationRefreshPlayerRemoving()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyRejoin", {
        Text = "Rejoin Notification", Default = true,
        Callback = function(v)
            _G.__EchoNotificationState.RejoinNotify = v and true or false
            _G.__EchoNotificationRefreshPlayerAdded()
            _G.__EchoNotificationRefreshPlayerRemoving()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyKick", {
        Text = "Kick notify", Default = true,
        Callback = function(v)
            _G.__EchoNotificationState.KickNotify = v and true or false
            _G.__EchoNotificationRefreshBlackhole()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyTargetJoin", {
        Text = "Target join Notification", Default = true,
        Callback = function(v)
            _G.__EchoNotificationState.TargetJoinNotify = v and true or false
            _G.__EchoNotificationRefreshPlayerAdded()
        end,
    }))

    table.insert(defaultSection, NotificationsTab:AddToggle("EchoNotifyPacket", {
        Text = "Packet Lag Notification", Default = false,
        Callback = function(v)
            _G.__EchoNotificationState.PacketNotify = v and true or false
            _G.__EchoNotificationRefreshPacket()
        end,
    }))

    local customizeSection = {}
    table.insert(customizeSection, NotificationsTab:AddLabel("<b>Customize Mode</b>"))
    table.insert(customizeSection, NotificationsTab:AddDivider())
    table.insert(customizeSection, NotificationsTab:AddLabel("<b>Enjoy Customizing :)</b>"))

    for _, slot in ipairs(EchoNotificationEventSlots) do
        local currentSound = config.PerEvent[slot.key] or slot.default

        table.insert(customizeSection, NotificationsTab:AddDropdown("EchoNotifySound_" .. slot.key, {
            Values = EchoNotificationSoundNames, Default = currentSound, Text = slot.label,
            Searchable = false, Multi = false,
            Callback = function(Value)
                config.PerEvent[slot.key] = Value
                pcall(function()
                    local soundId = EchoNotificationSounds[Value]
                    if not soundId then return end
                    local s = Instance.new("Sound")
                    s.Name = "EchoEventPreview_" .. slot.key
                    s.SoundId = soundId
                    s.Volume = math.clamp(tonumber(config.Volume) or 1, 0, 10)
                    s.PlayOnRemove = false
                    s.Parent = EchoSoundFolder
                    if not s.IsLoaded then
                        local deadline = tick() + 2
                        while not s.IsLoaded and tick() < deadline do task.wait(0.02) end
                    end
                    s:Play()
                    task.delay(6, function()
                        if s and s.Parent then s:Destroy() end
                    end)
                end)
            end,
        }))

end

    table.insert(customizeSection, NotificationsTab:AddDivider())
    table.insert(customizeSection, NotificationsTab:AddLabel("<b>Presets</b>"))

    _G.__EchoNotificationPresets = _G.__EchoNotificationPresets or {}
    _G.__EchoNotificationCurrentPreset = _G.__EchoNotificationCurrentPreset or "My Preset"

    table.insert(customizeSection, NotificationsTab:AddInput("EchoNotifyPresetName", {
        Text = "Preset Name", Default = _G.__EchoNotificationCurrentPreset,
        Placeholder = "Enter any preset name", Numeric = false, Finished = true,
        Callback = function(Value)
            local name = tostring(Value or "")
            if name:match("%S") then _G.__EchoNotificationCurrentPreset = name end
        end,
    }))

    local presetListDropdown = NotificationsTab:AddDropdown("EchoNotifyPresetList", {
        Values = {}, Default = nil, Text = "Saved Presets",
        Searchable = false, Multi = false, AllowNull = true,
        Callback = function(Value)
            if Value and tostring(Value) ~= "" and tostring(Value) ~= "---" then
                _G.__EchoNotificationCurrentPreset = tostring(Value)
                local opt = Options and Options.EchoNotifyPresetName
                if opt then pcall(function() opt:SetValue(_G.__EchoNotificationCurrentPreset) end) end
            end
        end,
    })
    table.insert(customizeSection, presetListDropdown)

    local function refreshPresetList(selectName)
        local names = {}
        for name in pairs(_G.__EchoNotificationPresets or {}) do table.insert(names, name) end
        table.sort(names, function(a, b) return tostring(a):lower() < tostring(b):lower() end)
        pcall(function() presetListDropdown:SetValues(names) end)
        if selectName and _G.__EchoNotificationPresets[selectName] then
            pcall(function() presetListDropdown:SetValue(selectName) end)
        end
    end

    local function readPresetNameFromUI()
        local name = _G.__EchoNotificationCurrentPreset or ""
        pcall(function()
            local opt = Options and Options.EchoNotifyPresetName
            if opt and opt.Value ~= nil and tostring(opt.Value) ~= "" then
                name = tostring(opt.Value)
            end
        end)
        return name:gsub("^%s+", ""):gsub("%s+$", "")
    end

    local function snapshotPreset()
        local data = { Events = {} }
        for _, slot in ipairs(EchoNotificationEventSlots) do
            local value = config.PerEvent[slot.key] or slot.default
            pcall(function()
                local opt = Options and Options["EchoNotifySound_" .. slot.key]
                if opt and opt.Value ~= nil and tostring(opt.Value) ~= "" then
                    value = tostring(opt.Value)
                end
            end)
            data.Events[slot.key] = value
        end
        return data
    end

    local function applyPreset(data)
        if type(data) ~= "table" or type(data.Events) ~= "table" then return false end
        for _, slot in ipairs(EchoNotificationEventSlots) do
            local value = data.Events[slot.key]
            if value then
                config.PerEvent[slot.key] = value
                pcall(function()
                    local opt = Options and Options["EchoNotifySound_" .. slot.key]
                    if opt then opt:SetValue(value) end
                end)
            end
        end
        return true
    end

    table.insert(customizeSection, NotificationsTab:AddButton({
        Text = "Save Preset",
        Func = function()
            local name = readPresetNameFromUI()
            if name == "" then
                Library:Notify({ Title = "Echo", Description = "Enter a preset name first.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
                return
            end
            if _G.__EchoNotificationPresets[name] then
                Library:Notify({ Title = "Echo", Description = name .. " already exists. Use Overwrite.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
                return
            end
            _G.__EchoNotificationPresets[name] = snapshotPreset()
            _G.__EchoNotificationCurrentPreset = name
            refreshPresetList(name)
            Library:Notify({ Title = "Echo", Description = name .. " saved.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
        end,
    }))

    table.insert(customizeSection, NotificationsTab:AddButton({
        Text = "Overwrite Preset",
        Func = function()
            local name = readPresetNameFromUI()
            if name == "" then return end
            if not _G.__EchoNotificationPresets[name] then return end
            _G.__EchoNotificationPresets[name] = snapshotPreset()
            _G.__EchoNotificationCurrentPreset = name
            refreshPresetList(name)
            Library:Notify({ Title = "Echo", Description = name .. " overwritten.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
        end,
    }))

    table.insert(customizeSection, NotificationsTab:AddButton({
        Text = "Load Preset",
        Func = function()
            local name = readPresetNameFromUI()
            if name == "" then return end
            local data = _G.__EchoNotificationPresets[name]
            if not data then return end
            applyPreset(data)
            _G.__EchoNotificationCurrentPreset = name
            Library:Notify({ Title = "Echo", Description = name .. " loaded.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
        end,
    }))

    table.insert(customizeSection, NotificationsTab:AddButton({
        Text = "Delete Preset",
        Func = function()
            local name = readPresetNameFromUI()
            if name == "" then return end
            if not _G.__EchoNotificationPresets[name] then return end
            _G.__EchoNotificationPresets[name] = nil
            refreshPresetList()
            Library:Notify({ Title = "Echo", Description = name .. " deleted.", Time = 3, Image = "rbxassetid://" .. tostring(Logo) })
        end,
    }))

    refreshPresetList()

    function _G.__EchoNotificationSyncMode()
        local isCustomize = (config.Mode == "Customize")
        for _, el in ipairs(defaultSection) do
            pcall(function()
                if el and el.SetVisible then el:SetVisible(not isCustomize) end
            end)
        end
        for _, el in ipairs(customizeSection) do
            pcall(function()
                if el and el.SetVisible then el:SetVisible(isCustomize) end
            end)
        end
    end

    _G.__EchoNotificationSyncMode()

    _G.__EchoApplyNotificationPlacement()
    _G.__EchoNotificationRefreshPlayerAdded()
    _G.__EchoNotificationRefreshPlayerRemoving()
    _G.__EchoNotificationRefreshPacket()
    _G.__EchoNotificationRefreshBlackhole()

    _G.__EchoNotificationsTabCleanup = function()
        for _, connection in pairs(_G.__EchoNotificationConnections or {}) do
            pcall(function() connection:Disconnect() end)
        end
        _G.__EchoNotificationConnections = {}

        _G.__EchoNotifyLastGlobal   = 0
        _G.__EchoNotifyKeyCooldowns = {}
        _G.__EchoNotifyDedupeCache  = {}
        _G.__EchoBlackholeWatch     = {}
        _G.__EchoLastPacketNotify   = 0

        if _G.__EchoNotificationSoundFolder and _G.__EchoNotificationSoundFolder.Parent then
            pcall(function() _G.__EchoNotificationSoundFolder:Destroy() end)
        end
        _G.__EchoNotificationSoundFolder = nil

        _G.__EchoNotificationSyncMode = nil
        _G.__EchoNotificationsTab = nil
    end

    pcall(function()
        Library:OnUnload(function()
            if _G.__EchoNotificationsTabCleanup then
                pcall(_G.__EchoNotificationsTabCleanup)
                _G.__EchoNotificationsTabCleanup = nil
            end
        end)
    end)
end -- closes Notifications do
end -- closes Visuals do

-- ═══════════════════════════════════════════════════════════════
--server tab
-- SERVER TAB
-- ═══════════════════════════════════════════════════════════════
do
    local ServerRight = Tabs.Server:AddRightGroupbox("Plot", "move")
-- ═══════════════════════════════════════════════════════════════
-- PLOT HELPERS (file-level, kept OUT of the Server tab scope)
-- ═══════════════════════════════════════════════════════════════
local PlotHelpers = (function()
    local PlotBreakerSelected = "Plot1"
    local PlotBarrierNoclipEnabled = false

    local PlotOptions = {
        "Plot1 (Green)", "Plot2 (Pink)", "Plot3 (Blue)",
        "Plot4 (Purple)", "Plot5 (Yellow)",
    }

    local PlotImages = {
        ["Plot1 (Green)"]  = "rbxassetid://0",
        ["Plot2 (Pink)"]   = "rbxassetid://0",
        ["Plot3 (Blue)"]   = "rbxassetid://0",
        ["Plot4 (Purple)"] = "rbxassetid://0",
        ["Plot5 (Yellow)"] = "rbxassetid://0",
    }

    local function getSelectedPlotNumber()
        local n = tostring(PlotBreakerSelected):match("Plot(%d)")
        return tonumber(n) or 1
    end

    local function getPlot(plotName)
        local plots = Workspace:FindFirstChild("Plots")
        return plots and plots:FindFirstChild(plotName)
    end

    -- Tracks plots that have actually been broken by the break cycle,
    -- so noclip toggling never falsely reports "Broken".
    local BrokenPlots = {}

    local function isPlotBroken(plotName)
        local plot = getPlot(plotName)
        if not plot then return false end
        local barrier = plot:FindFirstChild("Barrier")
        if not barrier then return false end

        -- Real broken = a PlotBarrier part is missing OR destroyed entirely.
        -- We only count it broken if the barrier model lost its parts,
        -- which is what the box-crate break cycle causes.
        local anyPart = false
        for _, obj in ipairs(barrier:GetDescendants()) do
            if obj:IsA("BasePart") and obj.Name == "PlotBarrier" then
                anyPart = true
                -- If a part exists and is still fully collidable AND not
                -- flagged as broken, treat it as intact.
                if obj.CanCollide and obj.CanQuery and obj.CanTouch then
                    return false
                end
            end
        end

        -- If no PlotBarrier parts remain, the plot is broken.
        if not anyPart then return true end

        -- If parts remain but at least one is non-collidable, only report
        -- broken if we explicitly flagged it as broken via the break cycle.
        return BrokenPlots[plotName] == true
    end

    local function breakPlotByName(plotName)
        local plot = getPlot(plotName)
        if not plot then return false end
        local barrier = plot:FindFirstChild("Barrier")
        if not barrier then return false end
        for _, obj in ipairs(barrier:GetDescendants()) do
            if obj:IsA("BasePart") and obj.Name == "PlotBarrier" then
                obj.CanCollide = false
                obj.CanQuery = false
                obj.CanTouch = false
            end
        end
        return true
    end

    local function breakSelectedPlotArea()
        local plot = getPlot("Plot" .. getSelectedPlotNumber())
        if not plot then return false end
        local area = plot:FindFirstChild("PlotArea")
        if area and area:IsA("BasePart") then
            area.CanCollide = false
            area.CanQuery = false
            area.CanTouch = false
            return true
        end
        return false
    end

    return {
        GetSelectedPlotNumber = getSelectedPlotNumber,
        GetPlot = getPlot,
        IsPlotBroken = isPlotBroken,
        BreakPlotByName = breakPlotByName,
        BreakSelectedPlotArea = breakSelectedPlotArea,
        PlotOptions = PlotOptions,
        PlotImages = PlotImages,
        GetSelected = function() return PlotBreakerSelected end,
        SetSelected = function(v) PlotBreakerSelected = v end,
        IsNoclipEnabled = function() return PlotBarrierNoclipEnabled end,
        SetNoclipEnabled = function(v) PlotBarrierNoclipEnabled = v end,
        SetPlotBroken = function(plotName, broken)
            BrokenPlots[plotName] = broken and true or nil
        end,
        ClearPlotBroken = function(plotName)
            BrokenPlots[plotName] = nil
        end,
    }
end)()

    -- Plot helper aliases
    local getSelectedPlotNumber = PlotHelpers.GetSelectedPlotNumber
    local getPlot = PlotHelpers.GetPlot
    local isPlotBroken = PlotHelpers.IsPlotBroken
    local breakPlotByName = PlotHelpers.BreakPlotByName
    local breakSelectedPlotArea = PlotHelpers.BreakSelectedPlotArea
    local PlotOptions = PlotHelpers.PlotOptions
    local PlotImages = PlotHelpers.PlotImages
    local PlotStatusLabel = nil

    local function updatePlotStatus()
        if not PlotStatusLabel then
            return
        end

        local plotName = "Plot" .. tostring(getSelectedPlotNumber())
        local broken = isPlotBroken(plotName)

        if broken then
            PlotStatusLabel:SetText("🟢 Plot Barrier Status: Broken")
        else
            PlotStatusLabel:SetText("🔴 Plot Barrier Status: Not Broken")
        end
    end

    -- ═══════════════════════════════════════════════════════════════
    -- LAG TABBOX — Line Lag / Packet Lag / Server Crash
    -- ═══════════════════════════════════════════════════════════════
    local LagTabbox        = Tabs.Server:AddLeftTabbox("Lag")
    local LineLagGroup     = LagTabbox:AddTab("Line Lag", "zap")
    local PacketLagGroup   = LagTabbox:AddTab("Packet Lag", "zap")
    local ServerCrashGroup = LagTabbox:AddTab("Server Crash", "server-off")

    -- ─────────────────────────────────────────────────────────────
    -- SHARED DETECTOR HELPERS
    -- ─────────────────────────────────────────────────────────────
    local DetKBThreshold = 5
    local DetByteThreshold = DetKBThreshold * 1024
    local DetCooldown = 15
    local DetLastNotify = 0

    local function detShortenString(str)
        if #str <= 80 then return str end
        return str:sub(1, 80) .. "... (+" .. (#str - 80) .. " chars)"
    end

    local function detCompressArgs(args)
        local seen, summary = {}, {}
        for _, v in ipairs(args) do
            local key
            if typeof(v) == "string" then
                key = "str:" .. detShortenString(v)
            elseif typeof(v) == "Instance" then
                key = "inst:" .. v.ClassName .. "(" .. v.Name .. ")"
            elseif typeof(v) == "table" then
                local count = 0
                for _ in pairs(v) do count = count + 1 end
                key = "tbl[" .. count .. "]"
            else
                key = typeof(v) .. ":" .. tostring(v)
            end
            seen[key] = (seen[key] or 0) + 1
        end
        for k, count in pairs(seen) do
            table.insert(summary, count > 1 and (k .. " x" .. count) or k)
        end
        return summary
    end

    local function detNotify(title, desc)
        _G.__EchoNotify(desc, "lag_detect_" .. tostring(title), 6, "PacketNotify")
    end

    local function detResolveSender(args)
        for _, v in ipairs(args) do
            if typeof(v) == "Instance" then
                if v:IsA("Player") then return v end
                if v:IsA("Model") or v:IsA("Tool") then
                    local plr = Players:GetPlayerFromCharacter(v)
                    if plr then return plr end
                end
                if v:IsA("BasePart") then
                    local model = v:FindFirstAncestorOfClass("Model")
                    if model then
                        local plr = Players:GetPlayerFromCharacter(model)
                        if plr then return plr end
                    end
                end
            end
        end
        return nil
    end

    local LagGrabEvents = ReplicatedStorage:FindFirstChild("GrabEvents")
    local ExtendGrabLine = LagGrabEvents and LagGrabEvents:FindFirstChild("ExtendGrabLine")

     -- ─────────────────────────────────────────────────────────────
    -- LINE LAG  (XOCU-style 1–10 level)
    -- Level L → (L * 100) sends per burst.
    --   L1   = 100 sends
    --   L5   = 500 sends (default)
    --   L10  = 1000 sends
    -- ─────────────────────────────────────────────────────────────
    local lagLineActive  = false
    local lagLineTask    = nil
    local lagLineLevel   = 5
    local lagLineAmount  = 500
    local lagLineDelay   = 0.05
    local lagLineDetect  = false
    local lagLineDetConn = nil

    local function lagLineUpdateAmount()
        local lvl = math.clamp(tonumber(lagLineLevel) or 5, 1, 10)
        lagLineAmount = lvl * 100
    end
    lagLineUpdateAmount()

    local function lagLineGetSpawn()
        return Workspace:FindFirstChild("SpawnLocation")
            or Workspace:FindFirstChild("Spawn")
            or (LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart"))
    end

    local function lagLineSendOnce()
        if not ExtendGrabLine then return end
        local loc = lagLineGetSpawn()
        if not loc then return end
        for _ = 1, math.max(1, math.floor(lagLineAmount)) do
            pcall(function()
                ExtendGrabLine:FireServer(loc, CFrame.new(0, 9e9, 0))
            end)
        end
    end

    local function lagLineStop()
        lagLineActive = false
        if lagLineTask then
            pcall(task.cancel, lagLineTask)
            lagLineTask = nil
        end
    end

    local function lagLineStart()
        lagLineStop()
        if not ExtendGrabLine then
            Library:Notify({ Title = "Echo", Description = "ExtendGrabLine not found.", Time = 3 })
            if Toggles.EchoLoopLineLag then
                pcall(function() Toggles.EchoLoopLineLag:SetValue(false) end)
            end
            return
        end
        lagLineActive = true
        lagLineTask = task.spawn(function()
            while lagLineActive do
                lagLineSendOnce()
                task.wait(lagLineDelay)
            end
        end)
    end

    local function lagLineDetectorStart()
        if lagLineDetConn or not ExtendGrabLine then return end
        lagLineDetConn = ExtendGrabLine.OnClientEvent:Connect(function(...)
            if not lagLineDetect then return end
            local args = { ... }

            local totalBytes = 0
            for _, v in ipairs(args) do
                if typeof(v) == "string" then totalBytes = totalBytes + #v end
            end
            if totalBytes < DetByteThreshold then return end

            local now = tick()
            if now - DetLastNotify < DetCooldown then return end
            DetLastNotify = now

            local sender = detResolveSender(args)
            local name
            if sender == LocalPlayer then
                name = sender.DisplayName .. " (" .. sender.Name .. ") [You]"
            elseif sender then
                name = sender.DisplayName .. " (" .. sender.Name .. ")"
            else
                name = "Unknown"
            end

            local mb = totalBytes / (1024 * 1024)
            local info = table.concat(detCompressArgs(args), " | ")
            detNotify("Line lag", string.format(
                "%s is Line lagging (%.2f MB) — %s",
                name, mb, info
            ))
        end)
    end

    local function lagLineDetectorStop()
        if lagLineDetConn then
            pcall(function() lagLineDetConn:Disconnect() end)
            lagLineDetConn = nil
        end
    end

    LineLagGroup:AddSlider("EchoLineLagLevel", {
        Text = "Line lag level (1-10)",
        Default = 5,
        Min = 1,
        Max = 10,
        Rounding = 0,
        Suffix = " lvl",
        Tooltip = "Level L → (L × 100) sends per burst. Level 10 = 1000 sends.",
        Callback = function(v)
            lagLineLevel = math.clamp(tonumber(v) or 5, 1, 10)
            lagLineUpdateAmount()
        end,
    })

    LineLagGroup:AddSlider("EchoLineLagDelay", {
        Text = "Delay Sec",
        Default = 0.05,
        Min = 0.01,
        Max = 1,
        Rounding = 2,
        Suffix = "s",
        Callback = function(v) lagLineDelay = math.clamp(tonumber(v) or 0.05, 0.01, 1) end,
    })

    LineLagGroup:AddToggle("EchoLoopLineLag", {
        Text = "Loop Line Lag",
        Default = false,
        Callback = function(v)
            if v then lagLineStart() else lagLineStop() end
        end,
    })

    LineLagGroup:AddToggle("EchoDetectLineLag", {
        Text = "Detect Line lag",
        Default = false,
        Callback = function(v)
            lagLineDetect = v and true or false
            if lagLineDetect then
                lagLineDetectorStart()
            else
                lagLineDetectorStop()
            end
        end,
    })

    LineLagGroup:AddButton({
        Text = "Line Lag Once",
        Func = function() lagLineSendOnce() end,
    })

    -- ─────────────────────────────────────────────────────────────
    -- PACKET LAG
    -- ─────────────────────────────────────────────────────────────
    local pktActive  = false
    local pktTask    = nil
    local pktSizeMB  = 1.5
    local pktDelay   = 0.15
    local pktDetect  = false
    local pktDetConn = nil

    local function pktBuildPayload()
        local bytes = math.floor(pktSizeMB * 1024 * 1024)
        if bytes <= 0 then bytes = 1024 end
        return string.rep("0", bytes)
    end

    local function pktSendOnce()
        if not ExtendGrabLine then return end
        local payload = pktBuildPayload()
        pcall(function()
            if ExtendGrabLine.ClassName == "RemoteFunction" then
                ExtendGrabLine:InvokeServer(payload)
            else
                ExtendGrabLine:FireServer(payload)
            end
        end)
    end

    local function pktStop()
        pktActive = false
        if pktTask then
            pcall(task.cancel, pktTask)
            pktTask = nil
        end
    end

    local function pktStart()
        pktStop()
        if not ExtendGrabLine then
            Library:Notify({ Title = "Echo", Description = "ExtendGrabLine not found.", Time = 3 })
            if Toggles.EchoLoopPacketLag then
                pcall(function() Toggles.EchoLoopPacketLag:SetValue(false) end)
            end
            return
        end
        pktActive = true
        pktTask = task.spawn(function()
            while pktActive do
                pktSendOnce()
                task.wait(pktDelay)
            end
        end)
    end

    local function pktDetectorStart()
        if pktDetConn or not ExtendGrabLine then return end
        pktDetConn = ExtendGrabLine.OnClientEvent:Connect(function(...)
            if not pktDetect then return end
            local args = { ... }

            local totalBytes = 0
            for _, v in ipairs(args) do
                if typeof(v) == "string" then totalBytes = totalBytes + #v end
            end
            if totalBytes < DetByteThreshold then return end

            local now = tick()
            if now - DetLastNotify < DetCooldown then return end
            DetLastNotify = now

            local sender = detResolveSender(args)
            local name
            if sender == LocalPlayer then
                name = sender.DisplayName .. " (" .. sender.Name .. ") [You]"
            elseif sender then
                name = sender.DisplayName .. " (" .. sender.Name .. ")"
            else
                name = "Unknown"
            end

            local mb = totalBytes / (1024 * 1024)
            local info = table.concat(detCompressArgs(args), " | ")
            detNotify("Packet Lag", string.format(
                "%s is Packet lagging (%.2f MB) — %s",
                name, mb, info
            ))
        end)
    end

    local function pktDetectorStop()
        if pktDetConn then
            pcall(function() pktDetConn:Disconnect() end)
            pktDetConn = nil
        end
    end

    PacketLagGroup:AddSlider("EchoPacketLagSize", {
        Text = "MB size",
        Default = 1.5,
        Min = 0.1,
        Max = 19,
        Rounding = 1,
        Suffix = " MB",
        Callback = function(v) pktSizeMB = math.clamp(tonumber(v) or 1.5, 0.1, 19) end,
    })

    PacketLagGroup:AddSlider("EchoPacketLagDelay", {
        Text = "Delay Sec",
        Default = 0.15,
        Min = 0.01,
        Max = 5,
        Rounding = 2,
        Suffix = "s",
        Callback = function(v) pktDelay = math.clamp(tonumber(v) or 0.15, 0.01, 5) end,
    })

    PacketLagGroup:AddToggle("EchoLoopPacketLag", {
        Text = "Loop Packet Lag",
        Default = false,
        Callback = function(v)
            if v then pktStart() else pktStop() end
        end,
    })

    PacketLagGroup:AddToggle("EchoDetectPacketLag", {
        Text = "Detect Packet lag",
        Default = false,
        Callback = function(v)
            pktDetect = v and true or false
            if pktDetect then
                pktDetectorStart()
            else
                pktDetectorStop()
            end
        end,
    })

    PacketLagGroup:AddButton({
        Text = "Packet Lag Once",
        Func = function() pktSendOnce() end,
    })

    -- ─────────────────────────────────────────────────────────────
    -- SERVER CRASH
    -- ─────────────────────────────────────────────────────────────
    local scActive  = false
    local scTask    = nil
    local scBurst   = 3
    local scSizeMB  = 0.75
    local scDelay   = 0.05
    local scDetect  = false
    local scDetConn = nil

    local function scBuildPayload()
        local bytes = math.floor(scSizeMB * 1024 * 1024)
        if bytes <= 0 then bytes = 1024 end
        return string.rep("0", bytes)
    end

    local function scSendOnce()
        if not ExtendGrabLine then return end
        local payload = scBuildPayload()
        for _ = 1, math.max(1, math.floor(scBurst)) do
            pcall(function()
                if ExtendGrabLine.ClassName == "RemoteFunction" then
                    ExtendGrabLine:InvokeServer(payload)
                else
                    ExtendGrabLine:FireServer(payload)
                end
            end)
        end
    end

    local function scStop()
        scActive = false
        if scTask then
            pcall(task.cancel, scTask)
            scTask = nil
        end
    end

    local function scStart()
        scStop()
        if not ExtendGrabLine then
            Library:Notify({ Title = "Echo", Description = "ExtendGrabLine not found.", Time = 3 })
            if Toggles.EchoLoopServerCrash then
                pcall(function() Toggles.EchoLoopServerCrash:SetValue(false) end)
            end
            return
        end
        scActive = true
        scTask = task.spawn(function()
            while scActive do
                scSendOnce()
                task.wait(scDelay)
            end
        end)
    end

    local function scDetectorStart()
        if scDetConn or not ExtendGrabLine then return end
        scDetConn = ExtendGrabLine.OnClientEvent:Connect(function(...)
            if not scDetect then return end
            local args = { ... }

            local totalBytes = 0
            for _, v in ipairs(args) do
                if typeof(v) == "string" then totalBytes = totalBytes + #v end
            end
            if totalBytes < 2 * 1024 * 1024 then return end

            local now = tick()
            if now - DetLastNotify < DetCooldown then return end
            DetLastNotify = now

            local sender = detResolveSender(args)
            local name
            if sender == LocalPlayer then
                name = sender.DisplayName .. " (" .. sender.Name .. ") [You]"
            elseif sender then
                name = sender.DisplayName .. " (" .. sender.Name .. ")"
            else
                name = "Unknown"
            end

            local mb = totalBytes / (1024 * 1024)
            detNotify("Server Crash", string.format(
                "%s is Server Crashing (%.2f MB payload)",
                name, mb
            ))
        end)
    end

    local function scDetectorStop()
        if scDetConn then
            pcall(function() scDetConn:Disconnect() end)
            scDetConn = nil
        end
    end

    ServerCrashGroup:AddSlider("EchoServerCrashBurst", {
        Text = "Burst Per Frame",
        Default = 3,
        Min = 1,
        Max = 30,
        Rounding = 0,
        Callback = function(v) scBurst = math.clamp(math.floor(tonumber(v) or 3), 1, 30) end,
    })

    ServerCrashGroup:AddSlider("EchoServerCrashSize", {
        Text = "MB Size",
        Default = 0.75,
        Min = 0.1,
        Max = 19,
        Rounding = 1,
        Suffix = " MB",
        Callback = function(v) scSizeMB = math.clamp(tonumber(v) or 0.75, 0.1, 19) end,
    })

    ServerCrashGroup:AddToggle("EchoLoopServerCrash", {
        Text = "Loop Server Crash",
        Default = false,
        Callback = function(v)
            if v then scStart() else scStop() end
        end,
    })

    ServerCrashGroup:AddToggle("EchoDetectServerCrash", {
        Text = "Detect Server Crash",
        Default = false,
        Callback = function(v)
            scDetect = v and true or false
            if scDetect then
                scDetectorStart()
            else
                scDetectorStop()
            end
        end,
    })

    ServerCrashGroup:AddButton({
        Text = "Server Crash Once",
        Func = function() scSendOnce() end,
    })

    -- ─────────────────────────────────────────────────────────────
    -- DESYNC
    -- ─────────────────────────────────────────────────────────────
    local DesyncGroup = Tabs.Server:AddLeftGroupbox("Desync", "shuffle")

    local DesyncEnabled = false
    local DesyncConnection = nil
    local DesyncCharacterConnection = nil
    local DesyncCharacter = nil
    local DesyncRoot = nil
    local DesyncTimer = 0
    local DesyncSyncing = false

    local DESYNC_RELEASE_TIME = 0.10
    local DESYNC_INTERVAL = 0.50

    local function RestoreDesyncRoot()
        local root = DesyncRoot
        if root and root.Parent then
            pcall(function()
                sethiddenproperty(root, "PhysicsRepRootPart", root)
            end)
        end
        DesyncSyncing = false
    end

    local function StopDesync()
        DesyncEnabled = false
        DesyncTimer = 0

        if DesyncConnection then
            pcall(function() DesyncConnection:Disconnect() end)
            DesyncConnection = nil
        end

        if DesyncCharacterConnection then
            pcall(function() DesyncCharacterConnection:Disconnect() end)
            DesyncCharacterConnection = nil
        end

        RestoreDesyncRoot()
        DesyncCharacter = nil
        DesyncRoot = nil
    end

    local function BindDesyncCharacter(character)
        if not DesyncEnabled or not character then return end

        DesyncCharacter = character
        DesyncRoot = character:FindFirstChild("HumanoidRootPart") or character:WaitForChild("HumanoidRootPart", 5)
        DesyncTimer = 0
        DesyncSyncing = false

        if DesyncRoot then
            pcall(function()
                sethiddenproperty(DesyncRoot, "PhysicsRepRootPart", DesyncRoot)
            end)
        end
    end

    local function StartDesync()
        StopDesync()
        DesyncEnabled = true

        BindDesyncCharacter(LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait())

        DesyncCharacterConnection = LocalPlayer.CharacterAdded:Connect(function(character)
            if not DesyncEnabled then return end
            BindDesyncCharacter(character)
        end)

        DesyncConnection = RunService.Heartbeat:Connect(function(deltaTime)
            if not DesyncEnabled then return end

            if not DesyncCharacter or not DesyncCharacter.Parent or not DesyncRoot or not DesyncRoot.Parent then
                local character = LocalPlayer.Character
                if character and character ~= DesyncCharacter then
                    BindDesyncCharacter(character)
                end
                return
            end

            if DesyncSyncing then return end

            DesyncTimer += deltaTime
            if DesyncTimer < DESYNC_INTERVAL then return end

            DesyncTimer = 0
            DesyncSyncing = true

            local root = DesyncRoot
            pcall(function()
                sethiddenproperty(root, "PhysicsRepRootPart", nil)
            end)

            task.delay(DESYNC_RELEASE_TIME, function()
                if not DesyncEnabled or DesyncRoot ~= root or not root.Parent then
                    if root and root.Parent then
                        pcall(function()
                            sethiddenproperty(root, "PhysicsRepRootPart", root)
                        end)
                    end
                    DesyncSyncing = false
                    return
                end

                pcall(function()
                    sethiddenproperty(root, "PhysicsRepRootPart", root)
                end)
                DesyncSyncing = false
            end)
        end)
    end

    DesyncGroup:AddToggle("EchoDesync", {
        Text = "Desync",
        Default = false,
        Callback = function(Value)
            if Value then
                StartDesync()
            else
                StopDesync()
            end
        end,
    })

-- ============================================================
-- GROUND BREAK (left side of Server tab)
-- ============================================================
do
    local GroundGroup = Tabs.Server:AddLeftGroupbox("Ground Break", "hammer")

    local RGS = ReplicatedStorage
    local WGS = Workspace
    local PGS = Players
    local LPGS = PGS.LocalPlayer
    local RGS_Run = game:GetService("RunService")

    local SpawnToyRemote  = RGS:WaitForChild("MenuToys"):WaitForChild("SpawnToyRemoteFunction")
    local DestroyToyGS    = RGS:WaitForChild("MenuToys"):WaitForChild("DestroyToy")
    local SetNetworkOwner = RGS:WaitForChild("GrabEvents"):WaitForChild("SetNetworkOwner")
    local DestroyGrabLine = RGS:WaitForChild("GrabEvents"):WaitForChild("DestroyGrabLine")
    local StickyPartEvent = RGS:WaitForChild("PlayerEvents"):WaitForChild("StickyPartEvent")

    local currentGrabbedHighlightGS = nil

    -- --------------------------------------------------------
    -- NOCLIP
    -- --------------------------------------------------------
    local GroundNoclipActive = false
    local GroundNoclipConn   = nil

    local function GroundNoclipStop()
        GroundNoclipActive = false
        if GroundNoclipConn then
            pcall(function() GroundNoclipConn:Disconnect() end)
            GroundNoclipConn = nil
        end
        -- Restore collision on our own character.
        local char = LPGS.Character
        if char then
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    pcall(function() part.CanCollide = true end)
                end
            end
        end
    end

    local function GroundNoclipStart()
        GroundNoclipStop()
        GroundNoclipActive = true
        GroundNoclipConn = RGS_Run.Stepped:Connect(function()
            if not GroundNoclipActive then return end
            local char = LPGS.Character
            if not char then return end
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = false
                end
            end
        end)
    end

    local function GetFolderGS()
        return getgenv().getCurrentToyFolderGS and getgenv().getCurrentToyFolderGS()
    end

    -- --------------------------------------------------------
    -- UI
    -- --------------------------------------------------------
    GroundGroup:AddToggle("EchoGroundNoclip", {
        Text = "Noclip",
        Default = false,
        Tooltip = "Disables collision on your character so you can pass through BaseGround and walls",
        Callback = function(v)
            if v then GroundNoclipStart() else GroundNoclipStop() end
        end,
    })

    GroundGroup:AddDivider()

    GroundGroup:AddButton({
        Text = "Find Closest BaseGround",
        Tooltip = "Locates the closest BaseGround part, highlights it, and copies its path",
        Func = function()
            local char = LPGS.Character
            local hrp  = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then
                Library:Notify({ Title = "Echo", Description = "No character!", Time = 3 })
                return
            end

            local baseGround = WGS:FindFirstChild("Map") and WGS.Map:FindFirstChild("BaseGround")
            if not baseGround then
                Library:Notify({ Title = "Echo", Description = "BaseGround not found!", Time = 3 })
                return
            end

            local children     = baseGround:GetChildren()
            local closestPart  = nil
            local closestIndex = nil
            local closestDist  = math.huge
            local myPos        = hrp.Position

            for i, child in ipairs(children) do
                if child:IsA("BasePart") then
                    local dist = (child.Position - myPos).Magnitude
                    if dist < closestDist then
                        closestDist  = dist
                        closestPart  = child
                        closestIndex = i
                    end
                end
            end

            if not closestPart then
                Library:Notify({ Title = "Echo", Description = "No BasePart found in BaseGround!", Time = 3 })
                return
            end

            local pos = closestPart.Position
            local info = string.format(
                "Name: %s\nIndex: [%d]\nPosition: Vector3.new(%.2f, %.2f, %.2f)\nDistance: %.2f studs\nAccess: workspace.Map.BaseGround:GetChildren()[%d]",
                closestPart.Name,
                closestIndex,
                pos.X, pos.Y, pos.Z,
                closestDist,
                closestIndex
            )

            pcall(function()
                if setclipboard then
                    setclipboard(info)
                elseif toclipboard then
                    toclipboard(info)
                end
            end)

            pcall(function()
                local highlight = Instance.new("Highlight")
                highlight.Name              = "ClosestBaseGroundESP"
                highlight.Adornee           = closestPart
                highlight.FillColor         = Color3.fromRGB(0, 255, 0)
                highlight.OutlineColor      = Color3.fromRGB(255, 255, 0)
                highlight.FillTransparency  = 0.4
                highlight.OutlineTransparency = 0
                highlight.DepthMode         = Enum.HighlightDepthMode.AlwaysOnTop
                highlight.Parent            = closestPart

                local billboard = Instance.new("BillboardGui")
                billboard.Name         = "ClosestBaseGroundLabel"
                billboard.Adornee      = closestPart
                billboard.Size         = UDim2.new(0, 300, 0, 80)
                billboard.StudsOffset  = Vector3.new(0, 5, 0)
                billboard.AlwaysOnTop  = true
                billboard.Parent       = closestPart

                local label = Instance.new("TextLabel")
                label.Size                 = UDim2.new(1, 0, 1, 0)
                label.BackgroundTransparency = 1
                label.Text                 = string.format("[%d] %s\n%.0f studs away", closestIndex, closestPart.Name, closestDist)
                label.TextColor3           = Color3.fromRGB(0, 255, 0)
                label.TextStrokeTransparency = 0
                label.TextStrokeColor3     = Color3.fromRGB(0, 0, 0)
                label.Font                 = Enum.Font.GothamBold
                label.TextScaled           = true
                label.Parent               = billboard

                local att0 = Instance.new("Attachment", hrp)
                local att1 = Instance.new("Attachment", closestPart)

                local beam = Instance.new("Beam")
                beam.Attachment0 = att0
                beam.Attachment1 = att1
                beam.Color       = ColorSequence.new(Color3.fromRGB(0, 255, 0), Color3.fromRGB(255, 255, 0))
                beam.Width0      = 0.5
                beam.Width1      = 0.5
                beam.FaceCamera  = true
                beam.LightEmission = 1
                beam.Transparency = NumberSequence.new(0)
                beam.Parent      = hrp

                task.delay(8, function()
                    if highlight then highlight:Destroy() end
                    if billboard then billboard:Destroy() end
                    if beam      then beam:Destroy() end
                    if att0      then att0:Destroy() end
                    if att1      then att1:Destroy() end
                end)
            end)

            Library:Notify({
                Title = "Echo",
                Description = "Closest: [" .. closestIndex .. "] " .. closestPart.Name .. " - copied!",
                Time = 5,
            })
        end,
    })

    pcall(function()
        Library:OnUnload(function()
            GroundNoclipStop()
        end)
    end)
end

    -- ═══════════════════════════════════════════════════════════════
    -- PLOT BARRIER BREAK CYCLE (strong method)
    -- ═══════════════════════════════════════════════════════════════
    local BARRIER_BOX_POS = Vector3.new(-542, -7, 67)

    local BreakBarrierActive = false
    local BreakBarrierMonitor = nil
    local BreakBarrierInv = nil
    local BreakBarrierDestroyToy = nil
    local BreakBarrierThread = nil

    pcall(function()
        local menu = ReplicatedStorage:FindFirstChild("MenuToys")
        BreakBarrierDestroyToy = menu and menu:FindFirstChild("DestroyToy")
    end)

    local function getBreakBarrierCharacter()
        local char = LocalPlayer.Character
        if not char then return nil, nil end
        return char, char:FindFirstChild("HumanoidRootPart")
    end

    local function getBreakBarrierInventory()
        return Workspace:FindFirstChild(LocalPlayer.Name .. "SpawnedInToys")
    end

    local function breakBarrierSpawnToy(name, cf)
        local E = _G._EchoDefence
        if E and E.spawnToy then
            return E.spawnToy(name, cf, Vector3.zero)
        end

        local menu = ReplicatedStorage:FindFirstChild("MenuToys")
        local remote = menu and menu:FindFirstChild("SpawnToyRemoteFunction")
        local inv = getBreakBarrierInventory()
        if not remote or not inv then return nil end

        local found
        local con = inv.ChildAdded:Connect(function(child)
            if child.Name == name then
                found = child
            end
        end)

        pcall(function()
            remote:InvokeServer(name, cf, Vector3.zero)
        end)

        local started = tick()
        while not found and tick() - started < 4 do
            task.wait(0.05)
        end

        if con then con:Disconnect() end
        return found
    end

    local function breakBarrierGrab(toy)
        if not toy then return false end
        local holdPart = toy:FindFirstChild("HoldPart")
        local holdFunc = holdPart and holdPart:FindFirstChild("HoldItemRemoteFunction")
        local char = LocalPlayer.Character
        if not holdFunc or not char then return false end

        local ok = pcall(function()
            holdFunc:InvokeServer(toy, char)
        end)
        return ok
    end

    local function breakBarrierDestroyBox(box)
        if box and BreakBarrierDestroyToy then
            pcall(function() BreakBarrierDestroyToy:FireServer(box) end)
        end
    end

    local function spawnBoxCrate()
        if not BreakBarrierActive then return false end

        BreakBarrierInv = BreakBarrierInv or getBreakBarrierInventory()
        local inv = BreakBarrierInv
        if not inv then return false end

        local existingBox = inv:FindFirstChild("BoxCrateWood")
        if existingBox then
            breakBarrierDestroyBox(existingBox)
            task.wait(0.08)
        end

        local box = breakBarrierSpawnToy("BoxCrateWood", CFrame.new(BARRIER_BOX_POS))
        if not box then return false end

        local boxPart = box:FindFirstChild("SoundPart") or box:WaitForChild("SoundPart", 1)
        if not boxPart then
            breakBarrierDestroyBox(box)
            return false
        end

        boxPart.CFrame = CFrame.new(BARRIER_BOX_POS)
        return true
    end

    local doBreakCycle

    local function startBoxMonitor()
        if BreakBarrierMonitor then
            BreakBarrierMonitor:Disconnect()
            BreakBarrierMonitor = nil
        end

        BreakBarrierInv = BreakBarrierInv or getBreakBarrierInventory()
        local inv = BreakBarrierInv
        if not inv then return end

        BreakBarrierMonitor = inv.ChildRemoved:Connect(function(child)
            if not BreakBarrierActive then return end
            if child and child.Name == "BoxCrateWood" then
                task.spawn(function()
                    task.wait(0.1)
                    if BreakBarrierActive then
                        task.spawn(function()
                            pcall(doBreakCycle)
                        end)
                    end
                end)
            end
        end)
    end

    doBreakCycle = function()
        if not BreakBarrierActive then return end

        local _, HRP = getBreakBarrierCharacter()
        if not HRP then return end

        local inv = getBreakBarrierInventory()
        if not inv then return end
        BreakBarrierInv = inv

        local pos = HRP.CFrame
        local burg = inv:FindFirstChild("FoodHamburger")
            or breakBarrierSpawnToy("FoodHamburger", HRP.CFrame * CFrame.new(5, 5, 20))

        task.wait(0.05)
        if not BreakBarrierActive then return end

        if burg then
            breakBarrierGrab(burg)
            task.wait(0.05)
            if not BreakBarrierActive then return end

            local waypoints = Workspace:FindFirstChild("Waypoints")
            local tudor = waypoints and waypoints:FindFirstChild("TudorHouse")
            if not tudor then return end

            HRP.CFrame = tudor.CFrame
            task.wait(0.05)
            if not BreakBarrierActive then return end

            if BreakBarrierDestroyToy then
                pcall(function() BreakBarrierDestroyToy:FireServer(burg) end)
            end
            task.wait(0.03)
            if not BreakBarrierActive then return end

            HRP.CFrame = pos
        end

        local waitStart = tick()
        while tick() - waitStart < 0.5 do
            if not BreakBarrierActive then return end
            task.wait(0.05)
        end

        if not BreakBarrierActive then return end

        if not inv:FindFirstChild("FoodHamburger") then
            local boxPlaced = spawnBoxCrate()

            if boxPlaced then
                startBoxMonitor()

                local stayStart = tick()
                local boxStayed = false

                while tick() - stayStart < 1.25 do
                    if not BreakBarrierActive then return end

                    local box = inv:FindFirstChild("BoxCrateWood")
                    if box then
                        boxStayed = true
                        local boxPart = box:FindFirstChild("SoundPart") or box:FindFirstChildWhichIsA("BasePart")
                        if boxPart then
                            if (boxPart.Position - BARRIER_BOX_POS).Magnitude > 5 then
                                boxPart.CFrame = CFrame.new(BARRIER_BOX_POS)
                            end
                        end
                    else
                        boxStayed = false
                        break
                    end
                    task.wait(0.1)
                end

                if boxStayed then
                    -- The box-crate break cycle succeeded — mark the plot
                    -- as actually broken so the status label can distinguish
                    -- a real break from a noclip toggle.
                    PlotHelpers.SetPlotBroken("Plot" .. getSelectedPlotNumber(), true)
                    BreakBarrierActive = false
                    return
                elseif BreakBarrierActive then
                    task.wait(0.1)
                    doBreakCycle()
                    return
                end
            elseif BreakBarrierActive then
                task.wait(0.1)
                doBreakCycle()
                return
            end
        elseif BreakBarrierActive then
            task.wait(0.1)
            doBreakCycle()
        end
    end

    local function stopBreakBarrier()
        BreakBarrierActive = false
        if BreakBarrierMonitor then
            BreakBarrierMonitor:Disconnect()
            BreakBarrierMonitor = nil
        end
        if BreakBarrierThread then
            pcall(task.cancel, BreakBarrierThread)
            BreakBarrierThread = nil
        end

        local inv = getBreakBarrierInventory()
        if inv then
            local box = inv:FindFirstChild("BoxCrateWood")
            if box then breakBarrierDestroyBox(box) end
        end
    end

    local function breakSelectedPlotBarrier()
        stopBreakBarrier()

        BreakBarrierInv = getBreakBarrierInventory()
        if not BreakBarrierInv then
            return false
        end

        BreakBarrierActive = true
        BreakBarrierThread = task.spawn(function()
            pcall(doBreakCycle)
        end)

        return true
    end

    -- Loops the STRONG barrier break across every plot 1..5.
    -- Waits for each barrier to actually flip before moving on (max 6s each).
    local function breakAllPlotBarriers()
        task.spawn(function()
            for i = 1, 5 do
                local plotName = "Plot" .. i
                PlotHelpers.SetSelected(plotName)
                updatePlotStatus()

                stopBreakBarrier()
                BreakBarrierInv = getBreakBarrierInventory()
                if not BreakBarrierInv then
                    task.wait(0.5)
                    continue
                end

                BreakBarrierActive = true
                BreakBarrierThread = task.spawn(function()
                    pcall(doBreakCycle)
                end)

                local deadline = tick() + 6
                while tick() < deadline do
                    task.wait(0.2)
                    if isPlotBroken(plotName) then break end
                    if not BreakBarrierActive then break end
                end

                stopBreakBarrier()
                updatePlotStatus()
                task.wait(0.2)
            end
            task.wait(0.3)
            updatePlotStatus()
        end)
    end

    local function breakAllPlots()
        for i = 1, 5 do
            breakPlotByName("Plot" .. i)
            task.wait(0.05)
        end
        task.wait(0.05)
        updatePlotStatus()
    end

    local function fixPlotByName(plotName)
        local plot = getPlot(plotName)
        if not plot then return false end

        local fixed = false
        local barrier = plot:FindFirstChild("Barrier")
        if barrier then
            for _, obj in ipairs(barrier:GetDescendants()) do
                if obj:IsA("BasePart") and obj.Name == "PlotBarrier" then
                    obj.CanCollide = true
                    obj.CanQuery = true
                    obj.CanTouch = true
                    fixed = true
                end
            end
        end

        local plotArea = plot:FindFirstChild("PlotArea")
        if plotArea and plotArea:IsA("BasePart") then
            plotArea.CanCollide = true
            plotArea.CanQuery = true
            plotArea.CanTouch = true
            fixed = true
        end

        -- Clear the broken flag so the status label correctly flips back
        -- to "Not Broken" once the barrier is restored.
        PlotHelpers.ClearPlotBroken(plotName)

        return fixed
    end

    local function fixSelectedPlot()
        local ok = fixPlotByName("Plot" .. getSelectedPlotNumber())
        task.wait(0.05)
        updatePlotStatus()
        return ok
    end

    function setBarrierNoclip(enabled)
        PlotBarrierNoclipEnabled = enabled and true or false

        local plots = Workspace:FindFirstChild("Plots")
        if plots then
            for _, plot in ipairs(plots:GetChildren()) do
                local barrier = plot:FindFirstChild("Barrier")
                if barrier then
                    for _, obj in ipairs(barrier:GetDescendants()) do
                        if obj:IsA("BasePart") and obj.Name == "PlotBarrier" then
                            obj.CanQuery = not PlotBarrierNoclipEnabled
                            obj.CanCollide = not PlotBarrierNoclipEnabled
                            obj.CanTouch = not PlotBarrierNoclipEnabled
                        end
                    end
                end
            end
        end

        pcall(function()
            Workspace:SetAttribute("PlotBarrierGrabable", PlotBarrierNoclipEnabled)
        end)
    end

    -- ═══════════════════════════════════════════════════════════════
    -- AUTO BREAK BARRIER STATE
    -- ═══════════════════════════════════════════════════════════════
    local AutoBreakBarrier = false
    local AutoBreakConnection = nil

    -- ═══════════════════════════════════════════════════════════════
    -- PLOT UI
    -- ═══════════════════════════════════════════════════════════════
    local PlotTabbox = ServerRight:AddTabbox("Plot")
    local PlotBarrierTab = PlotTabbox:AddTab("Plot Barrier")
    local PlotAreaTab = PlotTabbox:AddTab("Plot Area")

    PlotBarrierTab:AddDropdown("PlotBreakerSelect", {
        Text = "Select Plot",
        Values = PlotOptions,
        Default = "Plot1 (Green)",
        Multi = false,
        ValueImages = PlotImages,
        Callback = function(Value)
            PlotHelpers.SetSelected(
                Value:match("Plot(%d)") and ("Plot" .. Value:match("Plot(%d)")) or "Plot1"
            )
            updatePlotStatus()
        end,
    })

    PlotStatusLabel = PlotBarrierTab:AddLabel("🔴 Plot Barrier Status: Not Broken")

    -- ── Toggles first ──────────────────────────────────────────

    PlotBarrierTab:AddToggle("EchoAutoBreakBarrier", {
        Text = "Auto Break Barrier",
        Default = false,
        Tooltip = "Continuously runs the Break Plot Barrier button logic on the selected plot.",
        Callback = function(Value)
            AutoBreakBarrier = Value and true or false

            if AutoBreakConnection then
                pcall(function() AutoBreakConnection:Disconnect() end)
                AutoBreakConnection = nil
            end

            if not AutoBreakBarrier then return end

            AutoBreakConnection = RunService.Heartbeat:Connect(function()
                if not AutoBreakBarrier then return end
                -- Runs the exact same guarded function as the Break Plot
                -- Barrier button. Because the guard short-circuits when the
                -- barrier is already broken, the notify only fires once per
                -- break attempt and the loop doesn't spam the cycle.
                pcall(EchoBreakPlotBarrier)
            end)
        end,
    })
    PlotBarrierTab:AddToggle("BreakBarrierAdd", {
        Text = "Noclip Barrier",
        Default = false,
        Callback = function(Value)
            setBarrierNoclip(Value)
        end,
    })

    PlotBarrierTab:AddDivider()

    -- ── Buttons below ──────────────────────────────────────────

    -- Shared handler used by both the Break Plot Barrier button and the
    -- Auto Break Barrier toggle. Returns true if the break cycle was
    -- actually started, false if the barrier was already broken.
    local function EchoBreakPlotBarrier()
        local plotName = "Plot" .. getSelectedPlotNumber()

        if isPlotBroken(plotName) then
            Library:Notify({
                Title       = "Echo",
                Description = "Plot barrier already broken",
                Time        = 3,
            })
            return false
        end

        breakSelectedPlotBarrier()
        return true
    end

    PlotBarrierTab:AddButton({
        Text = "Break Plot Barrier",
        Func = function()
            EchoBreakPlotBarrier()
        end,
        DoubleClick = false,
    })
    PlotBarrierTab:AddButton({
        Text = "Break All Plot Barriers",
        Tooltip = "Loops the strong barrier break across Plot1–Plot5",
        Func = function()
            breakAllPlotBarriers()
        end,
        DoubleClick = false,
    })

    PlotAreaTab:AddButton({
        Text = "Break Plot Area",
        Func = function()
            breakSelectedPlotArea()
        end,
        DoubleClick = false,
    })

    PlotAreaTab:AddButton({
        Text = "Fix Plot Area",
        Func = function()
            fixSelectedPlot()
        end,
        DoubleClick = false,
    })

    PlotAreaTab:AddButton({
        Text = "Break All Plot Areas",
        Func = function()
            breakAllPlots()
        end,
        DoubleClick = false,
    })

    updatePlotStatus()

    -- ── Live status poller ─────────────────────────────────────
    task.spawn(function()
        while not Library.Unloaded do
            updatePlotStatus()
            task.wait(0.5)
        end
    end)

    -- ─────────────────────────────────────────────────────────────
    -- SOUND SPAM
    -- ─────────────────────────────────────────────────────────────
    do
        local SoundSpamGroup = Tabs.Server:AddRightGroupbox("Sound Spam", "volume-2")

        local SoundSpamState = {
            Keywords = {
                "kick", "dead", "yo", "ree", "mad", "uwu", "yay", "banana", "spook", "grr",
                "ew", "lol", "hehe", "hmm", "dad", "hold", "pew", "drink", "soda", "yeehaw",
                "fly", "ahem", "cough", "one", "two", "three", "four", "five", "mom", "yum",
                "sh", "aaa", "cry", "sing", "xd", "?"
            },
            Enabled = false,
            CurrentKeyword = nil,
            Delay = 0.1,
            LastTime = 0,
            Connection = nil,
            FailureCount = 0,
            MinDelay = 0.05,
        }
        SoundSpamState.CurrentKeyword = SoundSpamState.Keywords[1]

        local spamWarningLabel = SoundSpamGroup:AddLabel("<b>Warning: Low Values WILL Spike ping</b>")
        spamWarningLabel.Visible = false

        SoundSpamState.SetWarningVisible = function(visible)
            pcall(function() spamWarningLabel.Visible = visible end)
        end

        SoundSpamState.Stop = function()
            SoundSpamState.Enabled = false
            if SoundSpamState.Connection then
                pcall(function() SoundSpamState.Connection:Disconnect() end)
                SoundSpamState.Connection = nil
            end
        end

        SoundSpamState.SendKeyword = function(keyword)
            if type(keyword) ~= "string" or keyword == "" then return false end
            local ok = pcall(function()
                local textChat = game:GetService("TextChatService")
                local channels = textChat:FindFirstChild("TextChannels")
                local general = channels and channels:FindFirstChild("RBXGeneral")
                if general then
                    general:SendAsync("/clear " .. keyword)
                end
            end)
            if not ok then
                SoundSpamState.FailureCount = SoundSpamState.FailureCount + 1
                if SoundSpamState.FailureCount == 20 then
                    pcall(function()
                        Library:Notify({
                            Title = "Echo",
                            Description = "Sound Spam: chat is rejecting sends. Try a higher delay.",
                            Time = 4,
                        })
                    end)
                end
            end
            return ok
        end

        SoundSpamState.Start = function()
            SoundSpamState.Stop()
            SoundSpamState.Enabled = true
            SoundSpamState.FailureCount = 0
            SoundSpamState.LastTime = 0

            SoundSpamState.Connection = RunService.Heartbeat:Connect(function()
                if not SoundSpamState.Enabled then return end
                if type(SoundSpamState.CurrentKeyword) ~= "string" or SoundSpamState.CurrentKeyword == "" then return end

                local now = os.clock()
                if now - SoundSpamState.LastTime < SoundSpamState.Delay then return end
                SoundSpamState.LastTime = now
                SoundSpamState.SendKeyword(SoundSpamState.CurrentKeyword)
            end)
        end

        SoundSpamGroup:AddToggle("EchoSoundSpam", {
            Text = "Spam Sound",
            Default = false,
            Callback = function(Value)
                if Value then SoundSpamState.Start() else SoundSpamState.Stop() end
            end,
        })

        SoundSpamGroup:AddDropdown("EchoSoundSpamKeyword", {
            Text = "Select Sound",
            Values = SoundSpamState.Keywords,
            Default = SoundSpamState.Keywords[1],
            Multi = false,
            Callback = function(Value)
                if typeof(Value) == "table" then
                    SoundSpamState.CurrentKeyword = Value[1]
                else
                    SoundSpamState.CurrentKeyword = Value
                end
            end,
        })

        SoundSpamGroup:AddSlider("EchoSoundSpamDelay", {
            Text = "Send Delay",
            Default = 0.1,
            Min = SoundSpamState.MinDelay,
            Max = 5,
            Rounding = 2,
            Suffix = "s",
            Compact = true,
            Callback = function(Value)
                SoundSpamState.Delay = math.max(tonumber(Value) or 0.1, SoundSpamState.MinDelay)
                SoundSpamState.SetWarningVisible(SoundSpamState.Delay <= 0.15)
            end,
        })

        SoundSpamGroup:AddDivider()
    end

    -- ─────────────────────────────────────────────────────────────
    -- CLEANUP
    -- ─────────────────────────────────────────────────────────────
    pcall(function()
        Library:OnUnload(function()
            lagLineStop()
            lagLineDetectorStop()
            pktStop()
            pktDetectorStop()
            scStop()
            scDetectorStop()
            StopDesync()
            stopBreakBarrier()
            setBarrierNoclip(false)
            SoundSpamState.Stop()

            -- Clear all broken plot flags.
            for i = 1, 5 do
                PlotHelpers.ClearPlotBroken("Plot" .. i)
            end

            if AutoBreakConnection then
                pcall(function() AutoBreakConnection:Disconnect() end)
                AutoBreakConnection = nil
            end
            AutoBreakBarrier = false
        end)
    end)
end

--endserver tab

-- SETTINGS
--settings tab
local MenuGroup = Tabs.Settings:AddLeftGroupbox("Menu", "settings")
local BackgroundGroup = Tabs.Settings:AddRightGroupbox("Background", "palette")

MenuGroup:AddLabel("Menu bind"):AddKeyPicker("MenuKeybind", {
    Default = "RightShift",
    NoUI = true,
    Text = "Menu keybind",
    ChangedCallback = function(NewKey)
        if typeof(NewKey) == "EnumItem" then
            BindUIKey(NewKey)
            return
        end
        if type(NewKey) == "string" then
            local Success, Result = pcall(function() return Enum.KeyCode[NewKey] end)
            if Success and Result then
                BindUIKey(Result)
            end
        end
    end,
})

pcall(function()
    Library.ToggleKeybind = Options.MenuKeybind
end)

MenuGroup:AddToggle("KeybindMenuOpen", {
    Text = "Open Keybind Menu",
    Default = Library.KeybindFrame and Library.KeybindFrame.Visible or false,
    Callback = function(Value)
        if Library.KeybindFrame then
            Library.KeybindFrame.Visible = Value
        end
    end,
})

MenuGroup:AddToggle("CustomCursor", {
    Text = "Custom Cursor",
    Default = false,
    Callback = function(Value)
        Library.ShowCustomCursor = Value
        ApplyCursor()
    end,
})

MenuGroup:AddDropdown("NotificationSide", {
    Text = "Notification Side",
    Values = {"Left", "Right"},
    Default = "Right",
    Callback = function(Value)
        pcall(function() Library:SetNotifySide(Value) end)
    end,
})

MenuGroup:AddDropdown("DPIScale", {
    Text = "DPI Scale",
    Values = {"50%", "75%", "100%", "125%", "150%", "175%", "200%"},
    Default = "100%",
    Callback = function(Value)
        local CleanValue = tostring(Value):gsub("%%", "")
        local DPI = tonumber(CleanValue)
        if DPI then
            pcall(function() Library:SetDPIScale(DPI) end)
        end
    end,
})

MenuGroup:AddSlider("UICornerRadius", {
    Text = "Corner Radius",
    Default = 18,
    Min = 0,
    Max = 24,
    Rounding = 0,
    Callback = function(Value)
        pcall(function() Window:SetCornerRadius(Value) end)
    end,
})

MenuGroup:AddDivider()

ThemeManager:SetLibrary(Library)
ThemeManager:SetFolder("Echo")
ThemeManager:ApplyToTab(Tabs.Settings)

SaveManager:SetLibrary(Library)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({"MenuKeybind"})
SaveManager:SetFolder("Echo")
SaveManager:SetSubFolder("Main")
SaveManager:BuildConfigSection(Tabs.Settings)
SaveManager:LoadAutoloadConfig()

-- MOBILE TOGGLE
local MobileGui = nil

if UserInputService.TouchEnabled then
    MobileGui = Instance.new("ScreenGui")
    MobileGui.Name = "Echo_MobileToggle"
    MobileGui.ResetOnSpawn = false
    MobileGui.IgnoreGuiInset = true
    MobileGui.DisplayOrder = 999
    MobileGui.Parent = (_G.__EchoGetGuiParent and _G.__EchoGetGuiParent()) or CoreGui

    local Button = Instance.new("ImageButton")
    Button.Name = "ToggleButton"
    Button.Size = UDim2.fromOffset(60, 60)
    Button.Position = UDim2.new(1, -80, 1, -100)
    Button.AnchorPoint = Vector2.new(1, 1)
    Button.BackgroundColor3 = EchoColorFromRGB(40, 40, 40)
    Button.BackgroundTransparency = 0.25
    Button.BorderSizePixel = 0
    Button.Image = "rbxassetid://" .. tostring(Logo)
    Button.ScaleType = Enum.ScaleType.Fit
    Button.Parent = MobileGui

    local Corner = Instance.new("UICorner")
    Corner.CornerRadius = UDim.new(1, 0)
    Corner.Parent = Button

    local Dragging = false
    local DragStart
    local ButtonStart
    local DragInput

    Button.InputBegan:Connect(function(Input)
        if Input.UserInputType == Enum.UserInputType.Touch or Input.UserInputType == Enum.UserInputType.MouseButton1 then
            Dragging = true
            DragStart = Input.Position
            ButtonStart = Button.Position
            DragInput = Input
        end
    end)

    Button.InputChanged:Connect(function(Input)
        if Input.UserInputType == Enum.UserInputType.Touch or Input.UserInputType == Enum.UserInputType.MouseMovement then
            DragInput = Input
        end
    end)

    UserInputService.InputChanged:Connect(function(Input)
        if not Dragging or Input ~= DragInput then return end
        local Delta = Input.Position - DragStart
        Button.Position = UDim2.new(ButtonStart.X.Scale, ButtonStart.X.Offset + Delta.X, ButtonStart.Y.Scale, ButtonStart.Y.Offset + Delta.Y)
    end)

    UserInputService.InputEnded:Connect(function(Input)
        if Input == DragInput then Dragging = false end
    end)

    Button.Activated:Connect(function()
        if not Dragging then ToggleUI() end
    end)
end

--endsettings tab

-- TAB DIVIDERS
-- Add a clean divider at the end of each groupbox, including tabbox subtabs.
do
    local function AddTabDividers(Tab)
        if not Tab then return end

        if Tab.Groupboxes then
            for _, Groupbox in pairs(Tab.Groupboxes) do
                pcall(function()
                    if Groupbox and Groupbox.AddDivider then
                        Groupbox:AddDivider()
                    end
                end)
            end
        end

        if Tab.Tabboxes then
            for _, Tabbox in pairs(Tab.Tabboxes) do
                if Tabbox and Tabbox.Tabs then
                    for _, SubTab in pairs(Tabbox.Tabs) do
                        AddTabDividers(SubTab)
                    end
                end
            end
        end
    end

    for _, Tab in pairs(Tabs) do
        AddTabDividers(Tab)
    end
end

--endmisc tab

-- ============================================================
-- FIGURE TAB
-- ============================================================
do
    local FigureTab = Tabs.Figure

    local FigureGroup = FigureTab:AddLeftGroupbox("Figure Grab", "person-standing")
    local PoseGroup   = FigureTab:AddRightGroupbox("Poses", "move")

    -- --------------------------------------------------------
    -- POSE PRESETS (full set, keyed by internal name)
    -- --------------------------------------------------------
    local Presets = {
        Suspended = {
            HoldPosition = { X = 0,   Y = 0,    Z = -7.5 },
            HoldRotation = { X = 90,  Y = 0,    Z = 108  },
            LeftArmPosition  = { X = -1.5, Y = 1,    Z = -1   },
            LeftArmRotation  = { X = 283,  Y = 0,    Z = 0    },
            RightArmPosition = { X = 1.5,  Y = 0.5,  Z = 1    },
            RightArmRotation = { X = 270,  Y = 0,    Z = 0    },
            LeftLegPosition  = { X = 0.5,  Y = -1.5, Z = 0.5  },
            LeftLegRotation  = { X = 312,  Y = 0,    Z = 0    },
            RightLegPosition = { X = -0.5, Y = -1.5, Z = 0.5  },
            RightLegRotation = { X = 283,  Y = 0,    Z = 0    },
            HeadPosition     = { X = 0,    Y = 1.5,  Z = 0    },
            HeadRotation     = { X = 0,    Y = 0,    Z = 0    },
        },
        FaceDown = {
            HoldPosition = { X = 0,   Y = -1.5, Z = -12.5 },
            HoldRotation = { X = 272, Y = 0,    Z = 0     },
            LeftArmPosition  = { X = -1,  Y = 1,   Z = -0.5 },
            LeftArmRotation  = { X = 90,  Y = 0,   Z = 0    },
            RightArmPosition = { X = 1,   Y = 1,   Z = -0.5 },
            RightArmRotation = { X = 90,  Y = 0,   Z = 0    },
            LeftLegPosition  = { X = 1,   Y = -1,  Z = -0.5 },
            LeftLegRotation  = { X = 90,  Y = 0,   Z = 0    },
            RightLegPosition = { X = -1,  Y = -1,  Z = -0.5 },
            RightLegRotation = { X = 90,  Y = 0,   Z = 0    },
            HeadPosition     = { X = 0,   Y = 1,   Z = 1    },
            HeadRotation     = { X = 90,  Y = 0,   Z = 0    },
        },
        UpsideDown = {
            HoldPosition = { X = 0,   Y = -5.5, Z = -4 },
            HoldRotation = { X = 0,   Y = 0,    Z = 0  },
            LeftArmPosition  = { X = 1,   Y = 7.5, Z = 1.5 },
            LeftArmRotation  = { X = 0,   Y = 0,   Z = 0   },
            RightArmPosition = { X = 1,   Y = 6,   Z = 1.5 },
            RightArmRotation = { X = 0,   Y = 0,   Z = 0   },
            LeftLegPosition  = { X = 0.5, Y = 5,   Z = 1.5 },
            LeftLegRotation  = { X = 0,   Y = 0,   Z = 92  },
            RightLegPosition = { X = -0.5,Y = 5,   Z = 1.5 },
            RightLegRotation = { X = 0,   Y = 0,   Z = 90  },
            HeadPosition     = { X = 0,   Y = 0,   Z = 0   },
            HeadRotation     = { X = 0,   Y = 0,   Z = 0   },
        },
        HeadUp = {
            HoldPosition = { X = 1.5, Y = -8.5, Z = -1.5 },
            HoldRotation = { X = 0,   Y = 0,    Z = 0    },
            LeftArmPosition  = { X = 0,   Y = 0, Z = 0 },
            LeftArmRotation  = { X = 0,   Y = 0, Z = 0 },
            RightArmPosition = { X = 0,   Y = 0, Z = 0 },
            RightArmRotation = { X = 0,   Y = 0, Z = 0 },
            LeftLegPosition  = { X = 0,   Y = 0, Z = 0 },
            LeftLegRotation  = { X = 0,   Y = 0, Z = 0 },
            RightLegPosition = { X = 1.5, Y = 0, Z = 0 },
            RightLegRotation = { X = 0,   Y = 0, Z = 0 },
            HeadPosition     = { X = 0,   Y = 9, Z = 0 },
            HeadRotation     = { X = 0,   Y = 0, Z = 0 },
        },
        Dangling = {
            HoldPosition = { X = 0,   Y = -3,   Z = -6 },
            HoldRotation = { X = 270, Y = 0,    Z = 0  },
            LeftArmPosition  = { X = -1,  Y = 0.5, Z = 0   },
            LeftArmRotation  = { X = 180, Y = 0,   Z = 0   },
            RightArmPosition = { X = 1,   Y = 0.5, Z = 0   },
            RightArmRotation = { X = 180, Y = 0,   Z = 0   },
            LeftLegPosition  = { X = 0,   Y = -3,  Z = 0   },
            LeftLegRotation  = { X = 0,   Y = 0,   Z = 0   },
            RightLegPosition = { X = 0,   Y = -2,  Z = 0.5 },
            RightLegRotation = { X = 45,  Y = 0,   Z = 0   },
            HeadPosition     = { X = 0,   Y = 1.5, Z = -0.5},
            HeadRotation     = { X = 270, Y = 0,   Z = 0   },
        },
        Tilted = {
            HoldPosition = { X = 5.5, Y = 0.5, Z = -1.5 },
            HoldRotation = { X = 345, Y = 39,  Z = 0    },
            LeftArmPosition  = { X = 2,   Y = 0.5, Z = 0   },
            LeftArmRotation  = { X = 0,   Y = 43,  Z = 121 },
            RightArmPosition = { X = -2,  Y = 0,   Z = 0   },
            RightArmRotation = { X = 64,  Y = 112, Z = 0   },
            LeftLegPosition  = { X = -0.5,Y = -2,  Z = 0   },
            LeftLegRotation  = { X = 349, Y = 0,   Z = 360 },
            RightLegPosition = { X = 0.5, Y = -2,  Z = 0   },
            RightLegRotation = { X = 345, Y = 360, Z = 10  },
            HeadPosition     = { X = 0,   Y = 1.5, Z = 0   },
            HeadRotation     = { X = 0,   Y = 344, Z = 0   },
        },
        ArmsOut = {
            HoldPosition = { X = 0,   Y = -2,   Z = -10 },
            HoldRotation = { X = 90,  Y = 0,    Z = 0   },
            LeftArmPosition  = { X = -1.5,Y = 0,  Z = 0   },
            LeftArmRotation  = { X = 270, Y = 0,  Z = 315 },
            RightArmPosition = { X = 1.5, Y = 0,  Z = 0   },
            RightArmRotation = { X = 270, Y = 0,  Z = 45  },
            LeftLegPosition  = { X = -1,  Y = -1.5,Z = 0  },
            LeftLegRotation  = { X = 90,  Y = 0,  Z = 0   },
            RightLegPosition = { X = 1,   Y = -1.5,Z = 0  },
            RightLegRotation = { X = 90,  Y = 0,  Z = 0   },
            HeadPosition     = { X = 0,   Y = 1.5, Z = 0  },
            HeadRotation     = { X = 0,   Y = 0,  Z = 0   },
        },
        DogPose = {
            HoldPosition = { X = 0,   Y = -1.2, Z = -2.5 },
            HoldRotation = { X = -90, Y = 0,    Z = 0    },
            LeftArmPosition  = { X = -0.8, Y = 0.5,  Z = 0.5 },
            LeftArmRotation  = { X = 90,   Y = 0,    Z = 20  },
            RightArmPosition = { X = 0.8,  Y = 0.5,  Z = 0.5 },
            RightArmRotation = { X = 90,   Y = 0,    Z = -20 },
            LeftLegPosition  = { X = -0.6, Y = -1.2, Z = 0.5 },
            LeftLegRotation  = { X = 90,   Y = 0,    Z = 10  },
            RightLegPosition = { X = 0.6,  Y = -1.2, Z = 0.5 },
            RightLegRotation = { X = 90,   Y = 0,    Z = -10 },
            HeadPosition     = { X = 0,    Y = 1.3,  Z = -0.2},
            HeadRotation     = { X = 45,   Y = 0,    Z = 0   },
        },
    }

    -- --------------------------------------------------------
    -- FG state + API (ResetPose / ApplyPreset)
    -- --------------------------------------------------------
    local FRS = ReplicatedStorage
    local FGE = FRS:WaitForChild("GrabEvents")
    local FSetNetOwner     = FGE:WaitForChild("SetNetworkOwner")
    local FDestroyGrabLine = FGE:WaitForChild("DestroyGrabLine")

    local FG = {
        TargetName      = "",
        ActivePreset    = nil,
        Enabled         = false,
        Connection      = nil,
        TargetPlayer    = nil,
        TargetCharacter = nil,
        Config          = nil,
    }

    local ZERO = Vector3.zero

    local function BuildTargetCFrame(partName, torsoCF, cfg)
        local posKey, rotKey
        if partName == "Left Arm"  then posKey, rotKey = "LeftArmPosition",  "LeftArmRotation"
        elseif partName == "Right Arm" then posKey, rotKey = "RightArmPosition", "RightArmRotation"
        elseif partName == "Left Leg"  then posKey, rotKey = "LeftLegPosition",  "LeftLegRotation"
        elseif partName == "Right Leg" then posKey, rotKey = "RightLegPosition", "RightLegRotation"
        elseif partName == "Head"      then posKey, rotKey = "HeadPosition",     "HeadRotation"
        else return nil end
        local p = cfg[posKey]
        local r = cfg[rotKey]
        if not p or not r then return nil end
        return torsoCF * CFrame.new(p.X, p.Y, p.Z)
            * CFrame.Angles(math.rad(r.X), math.rad(r.Y), math.rad(r.Z))
    end

    local function GetPlayerList()
        local list = {}
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer then
                table.insert(list, p.DisplayName .. " (@" .. p.Name .. ")")
            end
        end
        table.sort(list)
        return list
    end

    local function UsernameFromDisplay(s)
        return s and s:match("@([%w_]+)")
    end

    function FG.ResetPose()
        FG.ActivePreset = nil
        FG.Config = nil
        if FG.Connection then
            pcall(function() FG.Connection:Disconnect() end)
            FG.Connection = nil
        end
        FG.Enabled = false
        if Toggles.EchoFigureGrab then
            pcall(function() Toggles.EchoFigureGrab:SetValue(false) end)
        end
    end

    function FG.ApplyPreset(presetKey)
        local preset = Presets[presetKey]
        if not preset then return false end
        FG.ActivePreset = presetKey
        FG.Config = preset
        -- If already grabbing, restart so the new pose takes effect immediately.
        if FG.Enabled then
            FG.Restart()
        end
        return true
    end

    function FG.Stop()
        FG.Enabled = false
        FG.TargetCharacter = nil
        FG.TargetPlayer = nil
        if FG.Connection then
            pcall(function() FG.Connection:Disconnect() end)
            FG.Connection = nil
        end
    end

    function FG.Restart()
        local targetName = FG.TargetName
        local presetKey  = FG.ActivePreset
        FG.Stop()
        if targetName ~= "" and presetKey then
            task.defer(function()
                if Toggles.EchoFigureGrab then
                    pcall(function() Toggles.EchoFigureGrab:SetValue(true) end)
                end
            end)
        end
    end

    function FG.Start()
        local target = Players:FindFirstChild(FG.TargetName)
        if not target then
            Library:Notify({ Title = "Echo", Description = "Target not found!", Time = 3 })
            if Toggles.EchoFigureGrab then pcall(function() Toggles.EchoFigureGrab:SetValue(false) end) end
            return
        end
        if not target.Character then
            Library:Notify({ Title = "Echo", Description = "Target has no character!", Time = 3 })
            if Toggles.EchoFigureGrab then pcall(function() Toggles.EchoFigureGrab:SetValue(false) end) end
            return
        end
        if not FG.Config then
            Library:Notify({ Title = "Echo", Description = "Pick a pose first.", Time = 3 })
            if Toggles.EchoFigureGrab then pcall(function() Toggles.EchoFigureGrab:SetValue(false) end) end
            return
        end

        FG.TargetPlayer = target
        FG.TargetCharacter = target.Character
        FG.Enabled = true

        FG.Connection = RunService.Heartbeat:Connect(function()
            if not FG.Enabled then return end

            local myChar = LocalPlayer.Character
            local myHRP  = myChar and myChar:FindFirstChild("HumanoidRootPart")
            if not myHRP then return end

            local t = FG.TargetPlayer
            if not t or not t.Parent then FG.Stop(); return end

            local char = t.Character
            if not char then return end
            FG.TargetCharacter = char

            local torso = char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
            if not torso then return end

            local cfg = FG.Config
            if not cfg then return end

            local holdCF = myHRP.CFrame
                * CFrame.new(cfg.HoldPosition.X, cfg.HoldPosition.Y, cfg.HoldPosition.Z)
                * CFrame.Angles(math.rad(cfg.HoldRotation.X), math.rad(cfg.HoldRotation.Y), math.rad(cfg.HoldRotation.Z))

            pcall(function()
                torso.CFrame = holdCF
                torso.AssemblyLinearVelocity  = ZERO
                torso.AssemblyAngularVelocity = ZERO
            end)

            for _, partName in ipairs({"Head", "Left Arm", "Right Arm", "Left Leg", "Right Leg"}) do
                local part = char:FindFirstChild(partName)
                if part then
                    local targetCF = BuildTargetCFrame(partName, torso.CFrame, cfg)
                    if targetCF then
                        pcall(function()
                            part.CFrame = targetCF
                            part.AssemblyLinearVelocity  = ZERO
                            part.AssemblyAngularVelocity = ZERO
                        end)
                    end
                end
            end

            pcall(function()
                FSetNetOwner:FireServer(torso, holdCF)
                FDestroyGrabLine:FireServer(torso)
            end)
        end)
    end

    -- --------------------------------------------------------
    -- UI
    -- --------------------------------------------------------
    FigureGroup:AddDropdown("EchoFigureTarget", {
        Text = "Target Player",
        Values = GetPlayerList(),
        Default = nil,
        Searchable = true,
        Multi = false,
        Callback = function(v)
            FG.TargetName = UsernameFromDisplay(v) or ""
        end,
    })

    FigureGroup:AddButton({
        Text = "Refresh Players",
        Func = function()
            pcall(function()
                Options.EchoFigureTarget:SetValues(GetPlayerList())
            end)
        end,
    })

    FigureGroup:AddDivider()

    FigureGroup:AddToggle("EchoFigureGrab", {
        Text = "Figure Grab",
        Default = false,
        Tooltip = "Grabs the target and holds them in the selected pose",
        Callback = function(v)
            if v then
                if FG.TargetName == "" then
                    Library:Notify({ Title = "Echo", Description = "Select a target player first.", Time = 3 })
                    pcall(function() Toggles.EchoFigureGrab:SetValue(false) end)
                    return
                end
                if not FG.Config then
                    Library:Notify({ Title = "Echo", Description = "Pick a pose first.", Time = 3 })
                    pcall(function() Toggles.EchoFigureGrab:SetValue(false) end)
                    return
                end
                FG.Start()
            else
                FG.Stop()
            end
        end,
    })

    -- --------------------------------------------------------
    -- POSE DROPDOWN (Reset + named poses)
    -- --------------------------------------------------------
    PoseGroup:AddDropdown("FigurePose", {
        Text = "Pose",
        Values = {
            "Reset",
            "DownwardDog",
            "BentL",
            "BrainHold",
            "UpsideDown",
            "Standing",
        },
        Default = "Reset",
        Multi = false,
        Callback = function(value)
            local map = {
                ["Reset"]       = "reset",
                ["DownwardDog"] = "DogPose",
                ["BentL"]       = "UpsideDown",
                ["BrainHold"]   = "HeadUp",
                ["UpsideDown"]  = "Dangling",
                ["Standing"]    = "Tilted",
            }
            local choice = map[value]
            if choice == "reset" then
                FG.ResetPose()
            elseif choice then
                FG.ApplyPreset(choice)
            end
        end,
    })

    Players.PlayerAdded:Connect(function()
        task.wait(0.5)
        pcall(function() Options.EchoFigureTarget:SetValues(GetPlayerList()) end)
    end)
    Players.PlayerRemoving:Connect(function()
        task.wait(0.5)
        pcall(function() Options.EchoFigureTarget:SetValues(GetPlayerList()) end)
    end)

    pcall(function()
        Library:OnUnload(function()
            FG.Stop()
        end)
    end)
end

-- FINAL CLEANUP
Library:OnUnload(function()
    pcall(function() if getgenv().__EchoClearCoconut then getgenv().__EchoClearCoconut() end end)
    getgenv().__EchoClearCoconut = nil
    pcall(function() ContextActionService:UnbindAction(ToggleAction) end)

    StopFlight()
    StopJerk()

    if WalkspeedConnection then
        pcall(function() WalkspeedConnection:Disconnect() end)
        WalkspeedConnection = nil
    end

    if JumpPowerConnection then
        pcall(function() JumpPowerConnection:Disconnect() end)
        JumpPowerConnection = nil
    end

    if NoclipConnection then
        pcall(function() NoclipConnection:Disconnect() end)
        NoclipConnection = nil
    end

    if InfiniteJumpConnection then
        pcall(function() InfiniteJumpConnection:Disconnect() end)
        InfiniteJumpConnection = nil
    end

    if MobileGui then
        pcall(function() MobileGui:Destroy() end)
        MobileGui = nil
    end

    pcall(function()
        LocalPlayer.CameraMode = OriginalCameraMode
        LocalPlayer.CameraMinZoomDistance = OriginalMinZoom
        LocalPlayer.CameraMaxZoomDistance = OriginalMaxZoom
        local Camera = Workspace.CurrentCamera
        if Camera then Camera.FieldOfView = 70 end
    end)

    pcall(function()
        UserInputService.MouseIconEnabled = false
        UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter
    end)

    print("[Echo] Unloaded.")
end)

--endserver tab

-- STARTUP
if _G.__EchoNotify then
    _G.__EchoNotify("Your presence has been heard. Welcome to Echo", "startup", 4, "Startup")
else
    Library:Notify({
        Title = "Echo",
        Description = "Your presence has been heard. Welcome to Echo",
        Time = 4,
        Image = "rbxassetid://" .. tostring(Logo),
    })
    pcall(function()
        if _G.__EchoPlayNotificationSound then
            _G.__EchoPlayNotificationSound("Startup")
        end
    end)
end

print("------------------------------------------")
print("                  ECHO")
print("------------------------------------------")
print("Version : " .. Version)
print("Profile : Static")
print("Logo    : " .. tostring(Logo))
print("Discord : " .. Discord)
print("Executor: " .. ExecutorName)
print("Player  : " .. LocalPlayer.Name)
print("Toggle  : RightShift")
print("------------------------------------------")
