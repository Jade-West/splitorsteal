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
    -- Hook the Connect method so that any future connections are replaced by a no-op
    local oldConnect = remote.OnClientEvent.Connect
    remote.OnClientEvent.Connect = function(self, callback)
        print("[BLOCKED] " .. remote.Name)
        -- Return a dummy connection that does nothing
        return {
            Connected = true,
            Disconnect = function() end
        }
    end
end

-- Find and block all matching remotes
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
        TabBar.Visible = false
        AutoFarmContainer.Visible = false
        AutoSellContainer.Visible = false
        MiscContainer.Visible = false
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
    end
end))

-- === SMART AUTO SELL SYSTEM ===
local autoSellEnabled = false
local sellThreshold = 0
local selectedRarities = {}
local sellQueue = {}
local isSelling = false
local rarityUIElements = {} 
local lastSellActivity = 0  -- to detect stuck selling

local allRarities = {
    "Common", "Uncommon", "Rare", "Epic", "Legendary", "Mythic", 
    "Event", "Secret", "Divine", "Cosmic", "Celestial", "Brainrot God"
}

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
MasterSellBtn.Size = UDim2.new(1, -10, 0, 35)
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
RarityGridContainer.LayoutOrder = 3
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

    -- Click the UnequipAll button to refresh inventory visuals
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
            -- Process one item at a time with a short delay, and retry if needed
            local queueSnapshot = {}
            for k, v in pairs(sellQueue) do queueSnapshot[k] = v end
            sellQueue = {}  -- clear global queue (will be repopulated next scan)
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
                    -- Reinsert it for next cycle, but only if it's still valid
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

            -- Build a set of IDs currently visible in the UI
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

            -- Remove queue entries that are no longer visible (already sold or vanished)
            for qId in pairs(sellQueue) do
                if not uiItemIds[qId] then
                    sellQueue[qId] = nil
                end
            end
        end)
        isScanning = false
    end

    -- Combined refresh: click UnequipAll, then scan
    local function fullRefreshCycle()
        clickUnequipAll()
        task.wait(0.3)  -- let UI update
        scanUIForItems()
        if next(sellQueue) ~= nil then
            processSellQueue()
        end
    end

    -- Listen to inventory updates (fires when new items arrive or inventory changes)
    if invRemote then
        registerConnection(invRemote.OnClientEvent:Connect(function()
            if not autoSellEnabled then return end
            task.wait(0.2)
            fullRefreshCycle()
        end))
    end

    -- Main periodic loop (every 5 seconds)
    registerThread(function()
        while task.wait(5) do
            if not autoSellEnabled then continue end
            -- If selling seems stuck for more than 15 seconds, force reset
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
local fpsConnections = {} -- store connections to disconnect later

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
    -- Graphics settings
    pcall(function()
        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        local Lighting = game:GetService("Lighting")
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 9e9
        
        -- Disable existing post-effects in Lighting
        for _, v in pairs(Lighting:GetDescendants()) do
            applyFpsBoostToObject(v)
        end

        -- Terrain
        local Terrain = workspace:FindFirstChildOfClass("Terrain")
        if Terrain then
            Terrain.WaterWaveSize = 0
            Terrain.WaterWaveSpeed = 0
            Terrain.WaterReflectance = 0
        end

        -- Clean up existing workspace objects
        for _, v in pairs(workspace:GetDescendants()) do
            applyFpsBoostToObject(v)
        end
    end)
end

local function toggleContinuousFPSBoost(state)
    fpsBoostActive = state
    if state then
        -- Run the initial heavy cleanup ONCE
        initialFpsBoost()
        
        -- Listen for new objects being added to workspace and Lighting
        local conn1 = workspace.DescendantAdded:Connect(function(obj)
            applyFpsBoostToObject(obj)
        end)
        local conn2 = game:GetService("Lighting").DescendantAdded:Connect(function(obj)
            applyFpsBoostToObject(obj)
        end)
        table.insert(fpsConnections, conn1)
        table.insert(fpsConnections, conn2)
    else
        -- Disconnect all event listeners
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
    
    { Tab = MiscTogglesFrame, Name = "Table ESP", IsToggleCustom = true, Action = toggleTableESP },
    { Tab = MiscTogglesFrame, Name = "Auto FPS Boost", IsToggleCustom = true, Action = toggleContinuousFPSBoost } 
}

-- ============================================================
-- === AUTO COUNTRY SELECT FOR WORLD CUP (Integrated into Autofarm)
-- ============================================================

-- Reference the remote that sets the player's country
local SetPlayerCountry = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("SetPlayerCountry", 10)

-- Country list with optional image ID
local countryOptions = {"USA", "Belgium", "Portugal", "England", "Brazil", "Argentina", "Spain", "France"}
local countryImageMap = {
    USA = "rbxassetid://76502735511314",
    Belgium = "rbxassetid://89365929372691",
    Portugal = "rbxassetid://130813352978477",
    England = "rbxassetid://130089205978603",
    Brazil = "rbxassetid://139029323188369",
    Argentina = "rbxassetid://117719698184156",
    Spain = "rbxassetid://72675024038912",
    France = "rbxassetid://121281535745152",
}

-- State variables
local autoCountryEnabled = false
local selectedCountry = "USA" -- default
local hasSelectedThisSession = false  -- prevent re-firing if already selected

-- Function to actually select the country
local function trySelectCountry()
    -- Check if World Cup is active
    local activeWeather = RS:FindFirstChild("ActiveWeather")
    if not activeWeather or activeWeather.Value ~= "WorldCup" then
        return false
    end

    -- Avoid spamming if we already selected one this session
    if hasSelectedThisSession then
        return true
    end

    pcall(function()
        -- Fire the main country selection remote
        if SetPlayerCountry then
            SetPlayerCountry:FireServer(selectedCountry)
        end

        -- Also update the global country image if that function exists
        if _G.SetWorldCupCountry and countryImageMap[selectedCountry] then
            _G.SetWorldCupCountry(countryImageMap[selectedCountry])
        end

        -- You could also fire the kit remote if you want the shirt/pants/trail
        local WorldCupKit = RS:WaitForChild("BrainrotsThings"):WaitForChild("Misc"):WaitForChild("Events"):WaitForChild("Player"):WaitForChild("WorldCupKit", 5)
        if WorldCupKit then
            -- Get the shirt/pants IDs from the country table (hardcoded from the original script)
            local countryData = {
                USA = {shirtId = 73296272711535, pantsId = 96913574679933},
                France = {shirtId = 112645314552199, pantsId = 129402259060225},
                Spain = {shirtId = 72691187130059, pantsId = 110152551144685},
                Argentina = {shirtId = 121144366710128, pantsId = 115213449183837},
                Brazil = {shirtId = 101302030512816, pantsId = 91549522860098},
                England = {shirtId = 107060255746071, pantsId = 11588596108},
                Portugal = {shirtId = 18837319900, pantsId = 113457248618129},
                Belgium = {shirtId = 113655360620987, pantsId = 104620621426486},
            }
            local cd = countryData[selectedCountry]
            if cd then
                WorldCupKit:FireServer(cd.shirtId, cd.pantsId)
            end
        end

        hasSelectedThisSession = true
        debugLog("Auto-selected country: " .. selectedCountry)
    end)
    return true
end

-- Reset selection flag when weather changes away from WorldCup (so it can fire again later)
if not _G._autoCountryWeatherListener then
    local weatherEvent = RS:WaitForChild("WeatherChanged", 10)
    if weatherEvent then
        local conn = weatherEvent.OnClientEvent:Connect(function(newWeather)
            if newWeather ~= "WorldCup" then
                hasSelectedThisSession = false
                debugLog("World Cup ended, resetting auto-country flag.")
            end
        end)
        registerConnection(conn)
        _G._autoCountryWeatherListener = true
    end
end

-- === BUILD THE UI (Button + Dropdown in AutoFarmContainer) ===

-- Main toggle button
local autoCountryBtn = Instance.new("TextButton")
autoCountryBtn.Size = UDim2.new(0, 143, 0, 34)
autoCountryBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
autoCountryBtn.Text = "Auto Country: OFF"
autoCountryBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
autoCountryBtn.Font = Enum.Font.GothamSemibold
autoCountryBtn.TextSize = 11
autoCountryBtn.LayoutOrder = 1000  -- Force to be last in the grid
autoCountryBtn.Parent = AutoFarmContainer

local autoBtnCorner = Instance.new("UICorner", autoCountryBtn)
autoBtnCorner.CornerRadius = UDim.new(0, 6)

local autoBtnStroke = Instance.new("UIStroke", autoCountryBtn)
autoBtnStroke.Color = Color3.fromRGB(50, 50, 60)

-- Country picker dropdown (like Play Again table selector)
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

-- Dropdown list (placed relative to MainFrame)
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

-- Populate country list
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

-- Show/hide dropdown when clicking the picker
registerConnection(countryPickerBtn.MouseButton1Click:Connect(function()
    countryListFrame.Visible = not countryListFrame.Visible
    if countryListFrame.Visible then
        local btnAbs = countryPickerBtn.AbsolutePosition
        local mainAbs = MainFrame.AbsolutePosition
        countryListFrame.Position = UDim2.new(0, btnAbs.X - mainAbs.X - 10, 0, btnAbs.Y - mainAbs.Y + countryPickerBtn.AbsoluteSize.Y + 2)
    end
end))

-- Toggle logic for the Auto Country button
registerConnection(autoCountryBtn.MouseButton1Click:Connect(function()
    autoCountryEnabled = not autoCountryEnabled
    if autoCountryEnabled then
        autoCountryBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 110)
        autoCountryBtn.TextColor3 = Color3.fromRGB(20, 35, 20)
        autoCountryBtn.Text = "Auto Country: ON"
        autoBtnStroke.Color = Color3.fromRGB(0, 200, 110)
        -- Start auto-select loop
        registerThread(function()
            while autoCountryEnabled do
                trySelectCountry()
                task.wait(5)
            end
        end)
    else
        autoCountryBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        autoCountryBtn.TextColor3 = Color3.fromRGB(150, 150, 160)
        autoCountryBtn.Text = "Auto Country: OFF"
        autoBtnStroke.Color = Color3.fromRGB(50, 50, 60)
    end
end))

-- ============================================================
-- END AUTO COUNTRY SELECT
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

-- Search Bar / Input
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

-- Permanent List Container (height reduced to 80)
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

-- Refresh List Hooks
registerConnection(ConfigNameBox.Focused:Connect(function() SearchStroke.Color = Color3.fromRGB(0, 200, 110) end))
registerConnection(ConfigNameBox.FocusLost:Connect(function() SearchStroke.Color = Color3.fromRGB(50, 50, 60) end))
registerConnection(ConfigNameBox:GetPropertyChangedSignal("Text"):Connect(function() refreshConfigList(ConfigNameBox.Text) end))

-- Central Loading Function
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

-- Action Buttons
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
        Rarities = selectedRarities
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

-- === DISCORD WEBHOOK LOGGING ===
local function sendWebhookLog()
    local req = (request or http_request or (syn and syn.request))
    if not req then 
        debugLog("HTTP requests not supported by this executor.")
        return 
    end

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
