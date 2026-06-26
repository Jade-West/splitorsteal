local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RS = game:GetService("ReplicatedStorage")
local VIM = game:GetService("VirtualInputManager")
local TS = game:GetService("TweenService")
local LocalPlayer = Players.LocalPlayer

-- === CONSOLE DEBUG LOGGER ===
local function debugLog(msg)
    print("[AUTO SELL DEBUG]: " .. tostring(msg))
end

-- === GHOST SCRIPT CLEANUP ===
if getgenv().SplitOrSteal_Connections then
    for _, conn in ipairs(getgenv().SplitOrSteal_Connections) do pcall(function() conn:Disconnect() end) end
end
if getgenv().SplitOrSteal_Threads then
    for _, thread in ipairs(getgenv().SplitOrSteal_Threads) do pcall(function() task.cancel(thread) end) end
end

getgenv().SplitOrSteal_Connections = {}
getgenv().SplitOrSteal_Threads = {}

local function registerConnection(conn)
    table.insert(getgenv().SplitOrSteal_Connections, conn)
    return conn
end

local function registerThread(func)
    local thread = task.spawn(func)
    table.insert(getgenv().SplitOrSteal_Threads, thread)
    return thread
end

-- === GUI SETUP ===
local guiParent = (gethui and gethui()) or game:GetService("CoreGui")
if not guiParent then guiParent = LocalPlayer:WaitForChild("PlayerGui") end

if guiParent:FindFirstChild("FancyRemoteGUI") then
    guiParent.FancyRemoteGUI:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "FancyRemoteGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = guiParent

registerConnection(UIS.InputBegan:Connect(function(input, gameProcessed)
    if not gameProcessed and input.KeyCode == Enum.KeyCode.RightControl then
        ScreenGui.Enabled = not ScreenGui.Enabled
    end
end))

-- === PREMIUM LOADING SCREEN ===
local LoadFrame = Instance.new("Frame")
LoadFrame.Size = UDim2.new(0, 0, 0, 0) -- Starts at 0 for bounce effect
LoadFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
LoadFrame.AnchorPoint = Vector2.new(0.5, 0.5)
LoadFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
LoadFrame.BorderSizePixel = 0
LoadFrame.ClipsDescendants = true
LoadFrame.Parent = ScreenGui

local LoadCorner = Instance.new("UICorner")
LoadCorner.CornerRadius = UDim.new(0, 12)
LoadCorner.Parent = LoadFrame

local LoadStroke = Instance.new("UIStroke")
LoadStroke.Color = Color3.fromRGB(0, 200, 110)
LoadStroke.Thickness = 2
LoadStroke.Transparency = 1
LoadStroke.Parent = LoadFrame

local WelcomeText = Instance.new("TextLabel")
WelcomeText.Size = UDim2.new(1, 0, 0.4, 0)
WelcomeText.Position = UDim2.new(0, 0, 0.15, 0)
WelcomeText.BackgroundTransparency = 1
WelcomeText.Text = "Initializing Hub..."
WelcomeText.TextColor3 = Color3.fromRGB(255, 255, 255)
WelcomeText.Font = Enum.Font.GothamBold
WelcomeText.TextSize = 18
WelcomeText.TextTransparency = 1
WelcomeText.Parent = LoadFrame

local UserText = Instance.new("TextLabel")
UserText.Size = UDim2.new(1, 0, 0.3, 0)
UserText.Position = UDim2.new(0, 0, 0.45, 0)
UserText.BackgroundTransparency = 1
UserText.Text = "Hello, " .. (LocalPlayer.DisplayName or LocalPlayer.Name)
UserText.TextColor3 = Color3.fromRGB(180, 180, 200)
UserText.Font = Enum.Font.GothamMedium
UserText.TextSize = 14
UserText.TextTransparency = 1
UserText.Parent = LoadFrame

local ProgressBarBg = Instance.new("Frame")
ProgressBarBg.Size = UDim2.new(0.8, 0, 0, 8)
ProgressBarBg.Position = UDim2.new(0.1, 0, 0.8, 0)
ProgressBarBg.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
ProgressBarBg.BorderSizePixel = 0
ProgressBarBg.Parent = LoadFrame
Instance.new("UICorner", ProgressBarBg).CornerRadius = UDim.new(1, 0)

local ProgressBarFill = Instance.new("Frame")
ProgressBarFill.Size = UDim2.new(0, 0, 1, 0)
ProgressBarFill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
ProgressBarFill.BorderSizePixel = 0
ProgressBarFill.Parent = ProgressBarBg
Instance.new("UICorner", ProgressBarFill).CornerRadius = UDim.new(1, 0)

local ProgressGradient = Instance.new("UIGradient")
ProgressGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 150, 200)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 255, 150))
})
ProgressGradient.Parent = ProgressBarFill

-- Loading Animation Sequence
TS:Create(LoadFrame, TweenInfo.new(0.6, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 280, 0, 140)}):Play()
TS:Create(LoadStroke, TweenInfo.new(0.5), {Transparency = 0}):Play()
task.wait(0.5)

TS:Create(WelcomeText, TweenInfo.new(0.5), {TextTransparency = 0}):Play()
task.wait(0.3)
TS:Create(UserText, TweenInfo.new(0.5), {TextTransparency = 0}):Play()
task.wait(0.3)

-- Smooth progress bar fill
local loadTween = TS:Create(ProgressBarFill, TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Size = UDim2.new(1, 0, 1, 0)})
loadTween:Play()
loadTween.Completed:Wait()

WelcomeText.Text = "Ready!"
WelcomeText.TextColor3 = Color3.fromRGB(0, 255, 150)
task.wait(0.4)

-- Fade out loading screen
TS:Create(LoadFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Size = UDim2.new(0, 0, 0, 0)}):Play()
TS:Create(LoadStroke, TweenInfo.new(0.4), {Transparency = 1}):Play()
task.wait(0.5)
LoadFrame:Destroy()

-- === PREMIUM UI MAIN STRUCTURE ===
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 260) 
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -130)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20) 
MainFrame.BackgroundTransparency = 1 -- Starts transparent for fade-in
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(60, 60, 70)
MainStroke.Thickness = 1.5
MainStroke.Transparency = 1
MainStroke.Parent = MainFrame

-- Fade in Main GUI
TS:Create(MainFrame, TweenInfo.new(0.5, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {BackgroundTransparency = 0.05}):Play()
TS:Create(MainStroke, TweenInfo.new(0.5), {Transparency = 0}):Play()

local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Size = UDim2.new(1, 0, 0, 35)
TopBar.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
TopBar.BorderSizePixel = 0
TopBar.Parent = MainFrame
Instance.new("UICorner", TopBar).CornerRadius = UDim.new(0, 10)

local TopBarExtension = Instance.new("Frame")
TopBarExtension.Size = UDim2.new(1, 0, 0, 10)
TopBarExtension.Position = UDim2.new(0, 0, 1, -10)
TopBarExtension.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
TopBarExtension.BorderSizePixel = 0
TopBarExtension.Parent = TopBar

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -60, 1, 0)
Title.Position = UDim2.new(0, 15, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "Split Or Steal Helper"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.TextSize = 14
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TopBar

local function createControlButton(text, color, posOffset)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 18, 0, 18)
    btn.Position = UDim2.new(1, posOffset, 0, 8)
    btn.BackgroundColor3 = color
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(25, 25, 30)
    btn.TextSize = 12
    btn.Font = Enum.Font.GothamBold
    btn.Parent = TopBar
    Instance.new("UICorner", btn).CornerRadius = UDim.new(1, 0)
    return btn
end

local MinimizeBtn = createControlButton("-", Color3.fromRGB(255, 170, 0), -50)
local CloseBtn = createControlButton("X", Color3.fromRGB(255, 75, 75), -25)

local BottomBar = Instance.new("Frame")
BottomBar.Name = "BottomBar"
BottomBar.Size = UDim2.new(1, 0, 0, 25)
BottomBar.Position = UDim2.new(0, 0, 1, -25)
BottomBar.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
BottomBar.BorderSizePixel = 0
BottomBar.Parent = MainFrame

local BottomStroke = Instance.new("Frame")
BottomStroke.Size = UDim2.new(1, 0, 0, 1)
BottomStroke.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
BottomStroke.BorderSizePixel = 0
BottomStroke.Parent = BottomBar

local PlaytimeLabel = Instance.new("TextLabel")
PlaytimeLabel.Size = UDim2.new(1, 0, 1, 0)
PlaytimeLabel.BackgroundTransparency = 1
PlaytimeLabel.Text = "Session Time: 00:00:00"
PlaytimeLabel.TextColor3 = Color3.fromRGB(180, 180, 200)
PlaytimeLabel.TextSize = 12
PlaytimeLabel.Font = Enum.Font.GothamMedium
PlaytimeLabel.Parent = BottomBar

local startTime = tick()
registerThread(function()
    while task.wait(1) do
        local elapsed = tick() - startTime
        local hours = math.floor(elapsed / 3600)
        local mins = math.floor((elapsed % 3600) / 60)
        local secs = math.floor(elapsed % 60)
        PlaytimeLabel.Text = string.format("Session Time: %02d:%02d:%02d", hours, mins, secs)
    end
end)

registerConnection(CloseBtn.MouseButton1Click:Connect(function()
    for _, conn in ipairs(getgenv().SplitOrSteal_Connections) do pcall(function() conn:Disconnect() end) end
    for _, thread in ipairs(getgenv().SplitOrSteal_Threads) do pcall(function() task.cancel(thread) end) end
    ScreenGui:Destroy()
end))

local dragging, dragStart, startPos
registerConnection(TopBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
        local moveConn, releaseConn
        
        moveConn = UIS.InputChanged:Connect(function(changeInput)
            if changeInput.UserInputType == Enum.UserInputType.MouseMovement or changeInput.UserInputType == Enum.UserInputType.Touch then
                local delta = changeInput.Position - dragStart
                MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
            end
        end)
        
        releaseConn = input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
                moveConn:Disconnect()
                releaseConn:Disconnect()
            end
        end)
    end
end))

-- === POPUP SYSTEM ===
local PopupFrame = Instance.new("Frame")
PopupFrame.Size = UDim2.new(1, 0, 1, 0)
PopupFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
PopupFrame.BackgroundTransparency = 0.2
PopupFrame.ZIndex = 100
PopupFrame.Visible = false
PopupFrame.Parent = MainFrame

local PopupBox = Instance.new("Frame")
PopupBox.Size = UDim2.new(0.8, 0, 0.5, 0)
PopupBox.Position = UDim2.new(0.1, 0, 0.25, 0)
PopupBox.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
PopupBox.ZIndex = 101
PopupBox.Parent = PopupFrame
Instance.new("UICorner", PopupBox).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", PopupBox).Color = Color3.fromRGB(60, 60, 70)

local PopupTitle = Instance.new("TextLabel")
PopupTitle.Size = UDim2.new(1, 0, 0.4, 0)
PopupTitle.BackgroundTransparency = 1
PopupTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
PopupTitle.Font = Enum.Font.GothamBold
PopupTitle.TextSize = 14
PopupTitle.TextWrapped = true
PopupTitle.ZIndex = 102
PopupTitle.Parent = PopupBox

local YesBtn = Instance.new("TextButton")
YesBtn.Size = UDim2.new(0.4, 0, 0.3, 0)
YesBtn.Position = UDim2.new(0.05, 0, 0.6, 0)
YesBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
YesBtn.Text = "YES"
YesBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
YesBtn.Font = Enum.Font.GothamBold
YesBtn.ZIndex = 102
YesBtn.Parent = PopupBox
Instance.new("UICorner", YesBtn).CornerRadius = UDim.new(0, 4)

local NoBtn = Instance.new("TextButton")
NoBtn.Size = UDim2.new(0.4, 0, 0.3, 0)
NoBtn.Position = UDim2.new(0.55, 0, 0.6, 0)
NoBtn.BackgroundColor3 = Color3.fromRGB(255, 75, 75)
NoBtn.Text = "NO"
NoBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
NoBtn.Font = Enum.Font.GothamBold
NoBtn.ZIndex = 102
NoBtn.Parent = PopupBox
Instance.new("UICorner", NoBtn).CornerRadius = UDim.new(0, 4)

local popupCallback = nil
registerConnection(YesBtn.MouseButton1Click:Connect(function()
    PopupFrame.Visible = false
    if popupCallback then popupCallback(true) end
end))
registerConnection(NoBtn.MouseButton1Click:Connect(function()
    PopupFrame.Visible = false
    if popupCallback then popupCallback(false) end
end))

local function ShowPopup(message, callback)
    PopupTitle.Text = message
    popupCallback = callback
    PopupFrame.Visible = true
end

-- === TAB NAVIGATION BAR ===
local TabBar = Instance.new("Frame")
TabBar.Name = "TabBar"
TabBar.Size = UDim2.new(1, -10, 0, 28)
TabBar.Position = UDim2.new(0, 5, 0, 40)
TabBar.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
TabBar.BorderSizePixel = 0
TabBar.Parent = MainFrame
Instance.new("UICorner", TabBar).CornerRadius = UDim.new(0, 6)

local TabListLayout = Instance.new("UIListLayout")
TabListLayout.FillDirection = Enum.FillDirection.Horizontal
TabListLayout.SortOrder = Enum.SortOrder.LayoutOrder
TabListLayout.Parent = TabBar

local function createTabButton(name, layoutOrder)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.333, 0, 1, 0)
    btn.LayoutOrder = layoutOrder
    btn.BackgroundTransparency = 1
    btn.Text = name
    btn.TextColor3 = Color3.fromRGB(150, 150, 160)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 11
    btn.Parent = TabBar
    
    local indicator = Instance.new("Frame")
    indicator.Size = UDim2.new(1, 0, 0, 2)
    indicator.Position = UDim2.new(0, 0, 1, -2)
    indicator.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
    indicator.BorderSizePixel = 0
    indicator.Visible = false
    indicator.Parent = btn
    
    return btn, indicator
end

local AutoFarmTabBtn, AutoFarmInd = createTabButton("AUTOFARM", 1)
local AutoSellTabBtn, AutoSellInd = createTabButton("AUTO SELL", 2)
local MiscTabBtn, MiscInd = createTabButton("MISC", 3)

local function createContainer(name)
    local container = Instance.new("ScrollingFrame")
    container.Name = name
    container.Size = UDim2.new(1, -10, 1, -100) 
    container.Position = UDim2.new(0, 5, 0, 72)
    container.BackgroundTransparency = 1
    container.BorderSizePixel = 0
    container.ScrollBarThickness = 3
    container.ScrollBarImageColor3 = Color3.fromRGB(100, 100, 110)
    container.AutomaticCanvasSize = Enum.AutomaticSize.Y 
    container.CanvasSize = UDim2.new(0, 0, 0, 0)
    container.Visible = false
    container.Parent = MainFrame
    return container
end

local AutoFarmContainer = createContainer("AutoFarmContainer")
local AutoSellContainer = createContainer("AutoSellContainer")
local MiscContainer = createContainer("MiscContainer")

for _, cont in ipairs({AutoFarmContainer, MiscContainer}) do
    local UIPadding = Instance.new("UIPadding")
    UIPadding.PaddingTop = UDim.new(0, 5)
    UIPadding.PaddingBottom = UDim.new(0, 5)
    UIPadding.PaddingLeft = UDim.new(0, 5)
    UIPadding.Parent = cont

    local UIGridLayout = Instance.new("UIGridLayout")
    UIGridLayout.CellSize = UDim2.new(0, 143, 0, 34) 
    UIGridLayout.CellPadding = UDim2.new(0, 8, 0, 8)
    UIGridLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIGridLayout.Parent = cont
end

local function switchTab(tabName)
    AutoFarmContainer.Visible = (tabName == "AutoFarm")
    AutoSellContainer.Visible = (tabName == "AutoSell")
    MiscContainer.Visible = (tabName == "Misc")

    AutoFarmTabBtn.TextColor3 = (tabName == "AutoFarm") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
    AutoSellTabBtn.TextColor3 = (tabName == "AutoSell") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
    MiscTabBtn.TextColor3 = (tabName == "Misc") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)

    AutoFarmInd.Visible = (tabName == "AutoFarm")
    AutoSellInd.Visible = (tabName == "AutoSell")
    MiscInd.Visible = (tabName == "Misc")
end

registerConnection(AutoFarmTabBtn.MouseButton1Click:Connect(function() switchTab("AutoFarm") end))
registerConnection(AutoSellTabBtn.MouseButton1Click:Connect(function() switchTab("AutoSell") end))
registerConnection(MiscTabBtn.MouseButton1Click:Connect(function() switchTab("Misc") end))
switchTab("AutoFarm")

-- === NEW MINIMIZE LOGIC ===
local minimized = false
local tweenInfo = TweenInfo.new(0.3, Enum.EasingStyle.Quart, Enum.EasingDirection.Out)
registerConnection(MinimizeBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        -- Hide everything to prevent buttons from blocking clicks
        TabBar.Visible = false
        AutoFarmContainer.Visible = false
        AutoSellContainer.Visible = false
        MiscContainer.Visible = false
        BottomBar.Visible = false
        
        TS:Create(MainFrame, tweenInfo, {Size = UDim2.new(0, 320, 0, 35)}):Play()
    else
        TS:Create(MainFrame, tweenInfo, {Size = UDim2.new(0, 320, 0, 260)}):Play()
        task.wait(0.2) -- Wait for the resize tween to finish before showing items
        
        TabBar.Visible = true
        BottomBar.Visible = true
        
        -- Restore only the tab that was currently active
        AutoFarmContainer.Visible = AutoFarmInd.Visible
        AutoSellContainer.Visible = AutoSellInd.Visible
        MiscContainer.Visible = MiscInd.Visible
    end
end))

-- [FIXED] Lag causing function: changed to execute once when clicked rather than spamming a loop
local function executeContinuousFPSBoost()
    pcall(function() settings().Rendering.QualityLevel = Enum.QualityLevel.Level01 end)
    local Lighting = game:GetService("Lighting")
    Lighting.GlobalShadows = false
    Lighting.FogEnd = 9e9
    
    for _, v in pairs(Lighting:GetDescendants()) do
        if v:IsA("PostEffect") or v:IsA("BlurEffect") or v:IsA("SunRaysEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("BloomEffect") or v:IsA("DepthOfFieldEffect") then
            v.Enabled = false
        end
    end

    local Terrain = workspace:FindFirstChildOfClass("Terrain")
    if Terrain then
        Terrain.WaterWaveSize = 0
        Terrain.WaterWaveSpeed = 0
        Terrain.WaterReflectance = 0
    end

    for _, v in pairs(workspace:GetDescendants()) do
        if v:IsA("BasePart") or v:IsA("MeshPart") then
            if v.Material ~= Enum.Material.SmoothPlastic then
                v.Material = Enum.Material.SmoothPlastic
                v.Reflectance = 0
                v.CastShadow = false
            end
        elseif v:IsA("Decal") or v:IsA("Texture") or v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Sparkles") or v:IsA("Smoke") or v:IsA("Fire") then
            v:Destroy()
        end
    end
end

local espObjects = {}
local function toggleTableESP(state)
    if state then
        for i = 1, 12 do
            local tableName = "Table" .. i
            local tableModel = workspace:FindFirstChild("Map") and workspace.Map:FindFirstChild("Tables") and workspace.Map.Tables:FindFirstChild(tableName)
            
            if tableModel then
                local targetPart = tableModel:IsA("Model") and tableModel.PrimaryPart or tableModel:FindFirstChildWhichIsA("BasePart") or tableModel
                if targetPart and targetPart:IsA("BasePart") then
                    local bgui = Instance.new("BillboardGui")
                    bgui.Name = "ESP_" .. tableName
                    bgui.AlwaysOnTop = true
                    bgui.Size = UDim2.new(0, 50, 0, 50)
                    bgui.StudsOffset = Vector3.new(0, 6, 0)
            
                    local txt = Instance.new("TextLabel")
                    txt.Size = UDim2.new(1, 0, 1, 0)
                    txt.BackgroundTransparency = 1
                    txt.Text = tostring(i)
                    txt.TextColor3 = Color3.fromRGB(0, 255, 150)
                    txt.Font = Enum.Font.GothamBlack
                    txt.TextScaled = true
                    txt.TextStrokeTransparency = 0
                    txt.Parent = bgui
                    bgui.Parent = targetPart
                    table.insert(espObjects, bgui)
                end
            end
        end
    else
        for _, bgui in ipairs(espObjects) do
            if bgui and bgui.Parent then bgui:Destroy() end
        end
        espObjects = {}
    end
end

-- === SMART AUTO SELL SYSTEM ===
local autoSellEnabled = false
local sellThreshold = 0
local selectedRarities = {}
local sellQueue = {}
local isSelling = false

local allRarities = {
    "Common", "Uncommon", "Rare", "Epic", "Legendary", "Mythic", 
    "Event", "Secret", "Divine", "Cosmic", "Celestial", "Brainrot God"
}

local AutoSellPadding = Instance.new("UIPadding", AutoSellContainer)
AutoSellPadding.PaddingTop = UDim.new(0, 5)
AutoSellPadding.PaddingBottom = UDim.new(0, 5)
AutoSellPadding.PaddingLeft = UDim.new(0, 5)
AutoSellPadding.PaddingRight = UDim.new(0, 5)

local AutoSellList = Instance.new("UIListLayout", AutoSellContainer)
AutoSellList.SortOrder = Enum.SortOrder.LayoutOrder
AutoSellList.Padding = UDim.new(0, 8)
AutoSellList.HorizontalAlignment = Enum.HorizontalAlignment.Center

local function parseValue(str)
    if not str or str == "" then return nil end
    str = tostring(str):lower():gsub(",", ""):gsub(" ", ""):gsub("%$", "")
    local multi = 1
    if str:match("k$") then multi = 1e3; str = str:gsub("k$", "")
    elseif str:match("m$") then multi = 1e6; str = str:gsub("m$", "")
    elseif str:match("b$") then multi = 1e9; str = str:gsub("b$", "")
    elseif str:match("t$") then multi = 1e12; str = str:gsub("t$", "") end
    local num = tonumber(str)
    return num and (num * multi) or nil
end

local MasterSellBtn = Instance.new("TextButton")
MasterSellBtn.Size = UDim2.new(1, 0, 0, 35)
MasterSellBtn.LayoutOrder = 1
MasterSellBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
MasterSellBtn.Text = "Smart Auto Sell: OFF"
MasterSellBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
MasterSellBtn.Font = Enum.Font.GothamBold
MasterSellBtn.TextSize = 13
MasterSellBtn.Parent = AutoSellContainer
Instance.new("UICorner", MasterSellBtn).CornerRadius = UDim.new(0, 6)
local MasterSellStroke = Instance.new("UIStroke", MasterSellBtn)
MasterSellStroke.Color = Color3.fromRGB(50, 50, 60)

registerConnection(MasterSellBtn.MouseButton1Click:Connect(function()
    autoSellEnabled = not autoSellEnabled
    if autoSellEnabled then
        MasterSellBtn.Text = "Smart Auto Sell: ON"
        MasterSellBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
        MasterSellBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
        MasterSellStroke.Color = Color3.fromRGB(0, 200, 110)
    else
        MasterSellBtn.Text = "Smart Auto Sell: OFF"
        MasterSellBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        MasterSellBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        MasterSellStroke.Color = Color3.fromRGB(50, 50, 60)
        sellQueue = {}
        isSelling = false
    end
end))

local ThresholdBox = Instance.new("TextBox")
ThresholdBox.Size = UDim2.new(1, 0, 0, 35)
ThresholdBox.LayoutOrder = 2
ThresholdBox.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
ThresholdBox.Text = ""
ThresholdBox.PlaceholderText = "Max Value (e.g., 1.5M, 2B, 10000)"
ThresholdBox.TextColor3 = Color3.fromRGB(255, 255, 255)
ThresholdBox.Font = Enum.Font.GothamSemibold
ThresholdBox.TextSize = 12
ThresholdBox.Parent = AutoSellContainer
Instance.new("UICorner", ThresholdBox).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", ThresholdBox).Color = Color3.fromRGB(50, 50, 60)

-- [FIXED] Safety logic for updating max threshold
registerConnection(ThresholdBox.FocusLost:Connect(function()
    local newVal = parseValue(ThresholdBox.Text)
    
    if not newVal then 
        ThresholdBox.Text = "Max Value: " .. string.format("%.0f", sellThreshold)
        return
    end

    ShowPopup("Are you sure you want to set the sell limit to " .. string.format("%.0f", newVal) .. "?", function(confirmed)
        if confirmed then
            sellThreshold = newVal
        end
        ThresholdBox.Text = "Max Value: " .. string.format("%.0f", sellThreshold)
    end)
end))

local RarityGridContainer = Instance.new("Frame")
RarityGridContainer.Size = UDim2.new(1, 0, 0, 0)
RarityGridContainer.LayoutOrder = 3
RarityGridContainer.BackgroundTransparency = 1
RarityGridContainer.AutomaticSize = Enum.AutomaticSize.Y
RarityGridContainer.Parent = AutoSellContainer

local RarityGrid = Instance.new("UIGridLayout", RarityGridContainer)
RarityGrid.CellSize = UDim2.new(0, 95, 0, 28)
RarityGrid.CellPadding = UDim2.new(0, 7, 0, 7)
RarityGrid.SortOrder = Enum.SortOrder.LayoutOrder

for i, rarity in ipairs(allRarities) do
    selectedRarities[rarity] = false

    local rBtn = Instance.new("TextButton")
    rBtn.LayoutOrder = i
    rBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    rBtn.Text = rarity
    rBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
    rBtn.Font = Enum.Font.GothamSemibold
    rBtn.TextSize = 11
    rBtn.Parent = RarityGridContainer
    Instance.new("UICorner", rBtn).CornerRadius = UDim.new(0, 4)
    local rStroke = Instance.new("UIStroke", rBtn)
    rStroke.Color = Color3.fromRGB(50, 50, 60)

    registerConnection(rBtn.MouseButton1Click:Connect(function()
        selectedRarities[rarity] = not selectedRarities[rarity]
        if selectedRarities[rarity] then
            rBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
            rBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            rStroke.Color = Color3.fromRGB(0, 150, 200)
        else
            rBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
            rBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
            rStroke.Color = Color3.fromRGB(50, 50, 60)
        end
    end))
end

registerThread(function()
    local invRemote
    pcall(function() 
        invRemote = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("InventoryUpdated")
    end)
    local sellRemote = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("SellItem")

    local function parseUITextValue(str)
        if type(str) ~= "string" then return 0 end
        str = str:lower():gsub("%$", ""):gsub("/s", ""):gsub(",", ""):gsub(" ", "")
        
        local multi = 1
        if str:match("k$") then multi = 1e3; str = str:gsub("k$", "")
        elseif str:match("m$") then multi = 1e6; str = str:gsub("m$", "")
        elseif str:match("b$") then multi = 1e9; str = str:gsub("b$", "")
        elseif str:match("t$") then multi = 1e12; str = str:gsub("t$", "") end
        
        local num = tonumber(str)
        return num and (num * multi) or 0
    end

    local function processSellQueue()
        if isSelling then return end
        isSelling = true
        
        registerThread(function()
            for itemId, _ in pairs(sellQueue) do
                if not autoSellEnabled then break end
                pcall(function() sellRemote:FireServer(itemId) end)
                sellQueue[itemId] = nil
                task.wait(0.15) 
            end
            isSelling = false
        end)
    end

    -- [FIXED] Added debounce and optimized queue to fix game freezing
    local isScanning = false
    local function scanUIForItems()
        if not autoSellEnabled or isScanning then return end
        isScanning = true
        
        pcall(function()
            local pGui = LocalPlayer:FindFirstChild("PlayerGui")
            if not pGui then return end
            
            local scrollFrame = pGui.FullGameGUIV2.InventoryHUD.Canvas.Scrollingframe
            local foundItems = false

            for _, itemFrame in ipairs(scrollFrame:GetChildren()) do
                if not itemFrame:IsA("UIListLayout") and not itemFrame:IsA("UIGridLayout") and not itemFrame:IsA("UIPadding") then
                    local content = itemFrame:FindFirstChild("Content")
                    if content then
                        local nameLabel = content:FindFirstChild("Name")
                        local rarityLabel = content:FindFirstChild("Rarity")
                        
                        if nameLabel and rarityLabel then
                            local itemValueStr = nameLabel.Text
                            local itemRarity = rarityLabel.Text
                            local parsedValue = parseUITextValue(itemValueStr)
                            
                            if selectedRarities[itemRarity] then
                                -- [FIXED] Checks that sellThreshold is greater than 0 before mass selling
                                if sellThreshold > 0 and parsedValue <= sellThreshold then
                                    local itemId = itemFrame.Name 
                                    if not sellQueue[itemId] then
                                        debugLog("QUEUED -> ID: " .. tostring(itemId))
                                        sellQueue[itemId] = true
                                        foundItems = true
                                    end
                                end
                            end
                        end
                    end
                end
            end
            
            if foundItems then
                processSellQueue()
            end
        end)
        isScanning = false
    end

    if invRemote then
        registerConnection(invRemote.OnClientEvent:Connect(function()
            if not autoSellEnabled then return end
            task.wait(0.2) 
            scanUIForItems()
        end))
    end
    
    registerThread(function()
        while task.wait(3) do -- Increased loop interval to prevent further lag
            if autoSellEnabled then
                scanUIForItems()
            end
        end
    end)
end)

-- === CONFIGURATION ===
local selectedPlayTable = "Table2"

local actionsConfig = {
    { Tab = AutoFarmContainer, Name = "Auto Split", Delay = 0.1, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Tables"):WaitForChild("DecisionRequest"):FireServer("Split") end },
    { Tab = AutoFarmContainer, Name = "Play Again", Delay = 0.1, IsPlayAgain = true, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("PlayAgainRequest"):FireServer(selectedPlayTable) end },
    { Tab = AutoFarmContainer, Name = "Auto Jump", Delay = 4, Action = function() VIM:SendKeyEvent(true, Enum.KeyCode.Space, false, game); task.wait(0.05); VIM:SendKeyEvent(false, Enum.KeyCode.Space, false, game) end },
    { Tab = AutoFarmContainer, Name = "Sell Celestial", Delay = 5, RequiresConfirm = true, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("QuickSellRarity"):FireServer("Celestial") end },
    { Tab = AutoFarmContainer, Name = "Sell Divine", Delay = 5, RequiresConfirm = true, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("QuickSellRarity"):FireServer("Divine") end },
    { Tab = AutoFarmContainer, Name = "Auto Collect", Delay = 5400, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("CollectAll"):FireServer() end },
    { Tab = AutoFarmContainer, Name = "Auto Trait", IsBurst = true, Action = function() 
        local merchantEvent = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("MerchantBuy")
        merchantEvent:FireServer("250KSlot")
        merchantEvent:FireServer("50MSlot")
        merchantEvent:FireServer("10BSlot")
        merchantEvent:FireServer("50MSlot")
    end },
    
    { Tab = MiscContainer, Name = "Table ESP", IsToggleCustom = true, Action = toggleTableESP },
    { Tab = MiscContainer, Name = "Auto FPS Boost", IsAction = true, Action = executeContinuousFPSBoost } 
}

for _, config in ipairs(actionsConfig) do
    local btn = Instance.new("TextButton")
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 12
    btn.Parent = config.Tab
    
    local btnCorner = Instance.new("UICorner", btn)
    btnCorner.CornerRadius = UDim.new(0, 6)

    local offColor = config.IsAction and Color3.fromRGB(180, 50, 50) or Color3.fromRGB(30, 30, 35)
    local onColor = Color3.fromRGB(0, 200, 110)
    local offTextColor = config.IsAction and Color3.fromRGB(255, 200, 200) or Color3.fromRGB(150, 150, 160)
    local onTextColor = Color3.fromRGB(20, 35, 20)
    
    local btnStroke = Instance.new("UIStroke")
    btnStroke.Color = Color3.fromRGB(50, 50, 60)
    btnStroke.Thickness = 1
    btnStroke.Parent = btn

    btn.BackgroundColor3 = offColor
    btn.Text = config.Name
    btn.TextColor3 = offTextColor

    if config.IsToggleCustom then
        local state = false
        registerConnection(btn.MouseButton1Click:Connect(function()
            state = not state
            btn.BackgroundColor3 = state and onColor or offColor
            btn.TextColor3 = state and onTextColor or offTextColor
            btnStroke.Color = state and onColor or Color3.fromRGB(50, 50, 60)
            pcall(function() config.Action(state) end)
        end))
        continue
    end

    if config.IsPlayAgain then
        local dropBtn = Instance.new("TextButton")
        dropBtn.Size = UDim2.new(0, 30, 1, -10)
        dropBtn.Position = UDim2.new(1, -35, 0, 5)
        dropBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
        dropBtn.Text = "T2"
        dropBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        dropBtn.Font = Enum.Font.GothamBold
        dropBtn.TextSize = 11
        dropBtn.ZIndex = 5
        Instance.new("UICorner", dropBtn).CornerRadius = UDim.new(0, 4)
        dropBtn.Parent = btn
        
        local listFrame = Instance.new("ScrollingFrame")
        listFrame.Size = UDim2.new(0, 45, 0, 110)
        listFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
        listFrame.BorderSizePixel = 0
        listFrame.ScrollBarThickness = 2
        listFrame.ZIndex = 50
        listFrame.Visible = false
        listFrame.Parent = MainFrame 
        
        local listLayout = Instance.new("UIListLayout", listFrame)
        listLayout.SortOrder = Enum.SortOrder.LayoutOrder
        
        for i = 1, 12 do
            local opt = Instance.new("TextButton")
            opt.Size = UDim2.new(1, 0, 0, 22)
            opt.BackgroundTransparency = 1
            opt.Text = tostring(i)
            opt.TextColor3 = Color3.fromRGB(200, 200, 200)
            opt.Font = Enum.Font.GothamSemibold
            opt.TextSize = 11
            opt.ZIndex = 51
            opt.Parent = listFrame
            
            registerConnection(opt.MouseButton1Click:Connect(function()
                selectedPlayTable = "Table" .. i
                dropBtn.Text = "T" .. i
                listFrame.Visible = false
            end))
        end
        
        registerConnection(dropBtn.MouseButton1Click:Connect(function()
            listFrame.Visible = not listFrame.Visible
            if listFrame.Visible then
                local absPos = dropBtn.AbsolutePosition
                local mainPos = MainFrame.AbsolutePosition
                listFrame.Position = UDim2.new(0, (absPos.X - mainPos.X) - 5, 0, (absPos.Y - mainPos.Y) + dropBtn.AbsoluteSize.Y + 2)
            end
        end))
    end

    local looping = false
    local activeThread = nil 

    local function executeToggleCode()
        if config.IsAction then
            pcall(config.Action)
            btn.Text = "EXECUTED!"
            btn.BackgroundColor3 = Color3.fromRGB(100, 100, 110)
            task.wait(1)
            btn.Text = config.Name
            btn.BackgroundColor3 = offColor
            return
        end

        looping = not looping
        if looping then
            btn.BackgroundColor3 = onColor
            btn.TextColor3 = onTextColor
            btnStroke.Color = onColor 
            
            activeThread = registerThread(function()
                while looping do
                    if config.IsBurst then
                        for burstCount = 1, 5 do
                            pcall(config.Action)
                            task.wait(0.05) 
                        end
                        for waitTick = 1, 1200 do task.wait(0.1) end
                    else
                        pcall(config.Action)
                        local ticksToWait = math.floor(config.Delay * 10)
                        for waitTick = 1, ticksToWait do task.wait(0.1) end
                    end
                end
            end)
        else
            btn.BackgroundColor3 = offColor
            btn.TextColor3 = offTextColor
            btnStroke.Color = Color3.fromRGB(50, 50, 60)
            
            if activeThread then
                task.cancel(activeThread)
                activeThread = nil
            end
        end
    end

    registerConnection(btn.MouseButton1Click:Connect(function()
        if config.RequiresConfirm and not looping then
            ShowPopup("Are you sure? This will sell ALL your " .. config.Name:gsub("Sell ", "") .. "s!", function(confirmed)
                if confirmed then executeToggleCode() end
            end)
        else
            executeToggleCode()
        end
    end))
end
