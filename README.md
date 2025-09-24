-- Redz Hub Ultra V5 – All Fixes & KRNL Compatible + Suppress Error
-- Upload this file to GitHub → raw URL → loadstring(...)() in executor

-- ===================================================================
-- A. EXECUTOR DETECTION & KEY SEND FALLBACK
-- ===================================================================
local isSyn       = typeof(syn) == "table"
local canKeyPress = typeof(keypress) == "function" and typeof(keyrelease) == "function"
local InputLib

if isSyn then
    InputLib = { sendKey = function(k,d) syn.input.sendKeyEvent(d, k, false) end }
elseif canKeyPress then
    InputLib = { sendKey = function(k,d) if d then keypress(k) else keyrelease(k) end end }
else
    local vu = game:GetService("VirtualUser")
    InputLib = {
        sendKey = function(k,d)
            vu:CaptureController()
            if d then vu:Button1Down(Vector2.new()) else vu:Button1Up(Vector2.new()) end
        end
    }
end

-- ===================================================================
-- B. SUPPRESS DEFAULT ERROR PANEL (SCRIPT SCAN & CLEAN)
-- ===================================================================
local ScriptContext = game:GetService("ScriptContext")
local CoreGui       = (gethui and gethui()) or game:GetService("Players").LocalPlayer:WaitForChild("PlayerGui")

ScriptContext.Error:Connect(function(msg, _stk, _src, _ln)
    warn("[RHU] Caught error:", msg)
    local function cleanse(gui)
        for _, o in ipairs(gui:GetChildren()) do
            if o:IsA("ScreenGui") or o:IsA("Frame") then
                if o.Name:lower():find("error") then o:Destroy() end
                cleanse(o)
            end
        end
    end
    cleanse(CoreGui)
end)

task.spawn(function()
    while true do
        task.wait(1)
        for _, gui in ipairs(CoreGui:GetChildren()) do
            if gui.Name:lower():find("error") then
                gui:Destroy()
            end
        end
    end
end)

-- ===================================================================
-- C. LOAD REDZ HUB LOADER (GUI & MENU GỐC)
-- ===================================================================
loadstring(game:HttpGet("https://raw.githubusercontent.com/YourUser/RedzHubUltra/main/main.luau"))()

-- ===================================================================
-- D. CACHE SERVICES & PLAYER OBJECTS
-- ===================================================================
local Players      = game:GetService("Players")
local RunService   = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

local LocalPlayer  = Players.LocalPlayer
local PlayerGui    = LocalPlayer:WaitForChild("PlayerGui")
local Character    = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
local HRP          = Character:WaitForChild("HumanoidRootPart")
local Humanoid     = Character:WaitForChild("Humanoid")

-- ===================================================================
-- E. ANTI-AFK & WALK SPEED CLAMP TO 350
-- ===================================================================
task.spawn(function()
    while true do
        task.wait(60)
        InputLib.sendKey(Enum.KeyCode.Space.Value, true)
        task.wait(1)
        InputLib.sendKey(Enum.KeyCode.Space.Value, false)
    end
end)

Humanoid.WalkSpeed = math.clamp(Humanoid.WalkSpeed, 16, 350)
Humanoid:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
    Humanoid.WalkSpeed = math.clamp(Humanoid.WalkSpeed, 16, 350)
end)

-- ===================================================================
-- F. SAFE ACTION WRAPPERS (SKILL & NPC INTERACT)
-- ===================================================================
local function safeUseSkill(keyEnum)
    task.wait(math.random(0.2, 0.5))
    InputLib.sendKey(keyEnum.Value, true)
    task.wait(math.random(0.3, 0.6))
    InputLib.sendKey(keyEnum.Value, false)
    task.wait(math.random(0.5, 1.0))
end

local function safeInteractWith(npc, remote, ...)
    local part = npc.PrimaryPart or npc:FindFirstChildWhichIsA("BasePart")
    if not part then return end
    TweenService:Create(HRP, TweenInfo.new(1, Enum.EasingStyle.Linear), {
        CFrame = part.CFrame * CFrame.new(0,0,3)
    }):Play()
    task.wait(1.2)
    part:FireTouchInterest(HRP,0,0)
    task.wait(0.5)
    local ok, res = pcall(function() return remote:InvokeServer(...) end)
    if not ok then warn("[RHU] interact error:", res) end
    return ok, res
end

-- ===================================================================
-- G. AUTOFARM BOSS (DEBOUNCE + TWEEN + SAFE SKILLS)
-- ===================================================================
local farmBusy = false
RunService.Heartbeat:Connect(function()
    if farmBusy then return end
    farmBusy = true

    local boss = workspace:FindFirstChild("Boss")
    if boss and boss:FindFirstChild("HumanoidRootPart") then
        TweenService:Create(HRP, TweenInfo.new(0.5, Enum.EasingStyle.Linear), {
            CFrame = boss.HumanoidRootPart.CFrame * CFrame.new(0,0,5)
        }):Play()
        task.wait(0.6)
        for _, k in ipairs({Enum.KeyCode.Z, Enum.KeyCode.X, Enum.KeyCode.C, Enum.KeyCode.V}) do
            safeUseSkill(k)
        end
    end

    farmBusy = false
end)

-- ===================================================================
-- H. ESP TỐI ƯU (THROTTLE 0.2s + DRAWING)
-- ===================================================================
local ESPCache = {}
local function registerESP(model, color)
    if ESPCache[model] then return end
    local box = Drawing.new("Square")
    box.Color       = color or Color3.new(1,0,0)
    box.Thickness   = 2
    box.Transparency= 1
    box.Visible     = false
    ESPCache[model] = box
end

registerESP(workspace:WaitForChild("Boss"),  Color3.new(1,0,0))
for _, f in ipairs(workspace:WaitForChild("Fruits"):GetChildren()) do
    registerESP(f, Color3.new(0,1,0))
end

task.spawn(function()
    while true do
        task.wait(0.2)
        for m, box in pairs(ESPCache) do
            if m.PrimaryPart then
                local sp, on = workspace.CurrentCamera:WorldToViewportPoint(m.PrimaryPart.Position)
                box.Visible = on
                if on then
                    box.Position = Vector2.new(sp.X-50, sp.Y-50)
                    box.Size     = Vector2.new(100,100)
                end
            end
        end
    end
end)

-- ===================================================================
-- I. SAFE TELEPORT (TWEEN)
-- ===================================================================
local function safeTeleport(pos)
    TweenService:Create(HRP, TweenInfo.new(1, Enum.EasingStyle.Linear), {
        CFrame = CFrame.new(pos)
    }):Play()
end

-- ===================================================================
-- J. GUI BASE – PlayerGui (AVOID CoreGui)
-- ===================================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name   = "RedzHubUltraV5"
ScreenGui.Parent = PlayerGui

local Frame = Instance.new("Frame", ScreenGui)
Frame.Size            = UDim2.new(0.3,0,0.5,0)
Frame.Position        = UDim2.new(0.35,0,0.25,0)
Frame.BackgroundColor3= Color3.fromRGB(30,30,30)

local btn = Instance.new("TextButton", Frame)
btn.Size             = UDim2.new(0.4,0,0.1,0)
btn.Position         = UDim2.new(0.05,0,0.05,0)
btn.Text             = "AutoFarm Boss"
btn.Font             = Enum.Font.FredokaOne
btn.TextColor3       = Color3.new(1,1,1)
btn.BackgroundColor3 = Color3.fromRGB(50,50,50)

-- ===================================================================
-- K. FINAL LOG
-- ===================================================================
print("✅ Redz Hub Ultra V5 Loaded – All Fixes & KRNL Compatible")
