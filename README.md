local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RS = game:GetService("ReplicatedStorage")
local VIM = game:GetService("VirtualInputManager")
local TS = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
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
MainFrame.Size = UDim2.new(0, 320, 0, 320) -- Increased height slightly for the cool config menu
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -160)
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
PopupBox.Size = UDim2.new(0.8, 0, 0.5, 0)
PopupBox.Position = UDim2.new(0.1, 0, 0.25, 0)
PopupBox.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
PopupBox.ZIndex = 101
PopupBox.Parent = PopupFrame
Instance.new("UICorner", PopupBox).CornerRadius = UDim.new(0, 8)
Instance.new("UIStroke", PopupBox).Color = Color3.fromRGB(60, 60, 70)

local PopupTitle = Instance.new("TextLabel")
PopupTitle.Size = UDim2.new(1, 0, 0.5, 0)
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

-- Setup Layouts for Farm and Sell Tabs
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

-- Structure for Misc Tab (Split between Toggles and Config)
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
        TS:Create(MainFrame, tweenInfo, {Size = UDim2.new(0, 320, 0, 320)}):Play()
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
            if foundItems then processSellQueue() end
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
        while task.wait(3) do
            if autoSellEnabled then scanUIForItems() end
        end
    end)
end)

-- === CPU/FPS BOOST LOOP ===
local fpsBoostActive = false
local function toggleContinuousFPSBoost(state)
    fpsBoostActive = state
    if state then
        registerThread(function()
            while fpsBoostActive do
                pcall(function()
                    settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
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
                end)
                task.wait(5) 
            end
        end)
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

-- Dropdown Menu (Animated)
local DropdownContainer = Instance.new("Frame", ConfigSection)
DropdownContainer.Size = UDim2.new(1, -10, 0, 0)
DropdownContainer.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
DropdownContainer.ClipsDescendants = true
Instance.new("UICorner", DropdownContainer).CornerRadius = UDim.new(0, 6)
local DropdownStroke = Instance.new("UIStroke", DropdownContainer)
DropdownStroke.Color = Color3.fromRGB(50, 50, 60)
DropdownStroke.Transparency = 1

local DropdownScroll = Instance.new("ScrollingFrame", DropdownContainer)
DropdownScroll.Size = UDim2.new(1, 0, 1, 0)
DropdownScroll.BackgroundTransparency = 1
DropdownScroll.BorderSizePixel = 0
DropdownScroll.ScrollBarThickness = 2
DropdownScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
DropdownScroll.CanvasSize = UDim2.new(0, 0, 0, 0)

local DropdownListLayout = Instance.new("UIListLayout", DropdownScroll)
DropdownListLayout.SortOrder = Enum.SortOrder.LayoutOrder

local function closeDropdown()
    TS:Create(DropdownContainer, TweenInfo.new(0.2, Enum.EasingStyle.Quart), {Size = UDim2.new(1, -10, 0, 0)}):Play()
    DropdownStroke.Transparency = 1
end

local function openDropdown()
    TS:Create(DropdownContainer, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(1, -10, 0, 100)}):Play()
    DropdownStroke.Transparency = 0
end

local configButtons = {}

local function refreshConfigList(filterText)
    -- Clear old list
    for _, btn in ipairs(configButtons) do btn:Destroy() end
    configButtons = {}
    
    if not listfiles then return end
    
    pcall(function()
        if not isfolder(CONFIG_FOLDER) then makefolder(CONFIG_FOLDER) end
        local files = listfiles(CONFIG_FOLDER)
        
        for _, path in ipairs(files) do
            -- Extract filename from path
            local fileName = string.match(path, "[^\\/]+$")
            if fileName and fileName:match("%.json$") then
                local cleanName = fileName:gsub("%.json$", "")
                
                -- Apply search filter
                if not filterText or filterText == "" or string.lower(cleanName):match(string.lower(filterText)) then
                    local btn = Instance.new("TextButton", DropdownScroll)
                    btn.Size = UDim2.new(1, 0, 0, 25)
                    btn.BackgroundTransparency = 1
                    btn.Text = "  " .. cleanName
                    btn.TextColor3 = Color3.fromRGB(200, 200, 200)
                    btn.Font = Enum.Font.GothamMedium
                    btn.TextSize = 12
                    btn.TextXAlignment = Enum.TextXAlignment.Left
                    
                    registerConnection(btn.MouseButton1Click:Connect(function()
                        ConfigNameBox.Text = cleanName
                        closeDropdown()
                    end))
                    
                    registerConnection(btn.MouseEnter:Connect(function() btn.TextColor3 = Color3.fromRGB(0, 200, 110) end))
                    registerConnection(btn.MouseLeave:Connect(function() btn.TextColor3 = Color3.fromRGB(200, 200, 200) end))
                    
                    table.insert(configButtons, btn)
                end
            end
        end
    end)
end

-- Hook up Search/Dropdown triggers
registerConnection(ConfigNameBox.Focused:Connect(function()
    refreshConfigList(ConfigNameBox.Text)
    openDropdown()
    SearchStroke.Color = Color3.fromRGB(0, 200, 110)
end))

registerConnection(ConfigNameBox.FocusLost:Connect(function(enterPressed)
    SearchStroke.Color = Color3.fromRGB(50, 50, 60)
    -- Small delay so click registers before closing
    task.delay(0.15, function() closeDropdown() end) 
end))

registerConnection(ConfigNameBox:GetPropertyChangedSignal("Text"):Connect(function()
    refreshConfigList(ConfigNameBox.Text)
end))

-- Buttons Container
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
        ShowPopup("Your executor does not support saving files!", function() end)
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
        refreshConfigList() -- Update list after saving
    end)
end))

registerConnection(LoadBtn.MouseButton1Click:Connect(function()
    if not readfile then return end
    local name = ConfigNameBox.Text
    if name == "" then name = "Default" end
    
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
        ShowPopup("Loaded Config: " .. name, function() end)
    end)
end))
