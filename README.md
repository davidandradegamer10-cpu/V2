
--==================================================
-- DAVID HUB V2 | GREEN EDITION
-- Egg Finder / Settings / Auto Hop
--==================================================

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local UIS = game:GetService("UserInputService")
local LP = Players.LocalPlayer

--==================================================
-- CONFIGURAÇÕES
--==================================================

local Config = {
    MinB = 1,
    MaxB = 10,
    UIScale = 1,
    AutoHop = false,
    Delay = 2,
    Rarity = "All",
    ScanInterval = 5,
    MaxResults = 5,
}

local RARITIES = {
    "All", "Common", "Rare", "Epic",
    "Legendary", "Mythic", "Divine"
}

local COLORS = {
    Background = Color3.fromRGB(10, 17, 14),
    Panel = Color3.fromRGB(17, 29, 22),
    Card = Color3.fromRGB(25, 43, 32),
    Green = Color3.fromRGB(46, 204, 113),
    GreenDark = Color3.fromRGB(20, 125, 65),
    Text = Color3.fromRGB(240, 255, 245),
    Muted = Color3.fromRGB(145, 170, 150),
    Red = Color3.fromRGB(230, 75, 75),
}

local RARITY_COLOR = {
    Common = Color3.fromRGB(200, 200, 200),
    Rare = Color3.fromRGB(80, 170, 255),
    Epic = Color3.fromRGB(190, 90, 255),
    Legendary = Color3.fromRGB(255, 190, 40),
    Mythic = Color3.fromRGB(255, 80, 120),
    Divine = Color3.fromRGB(170, 70, 255),
}

--==================================================
-- VARIÁVEIS
--==================================================

local Entries = {}
local Seen = {}
local CurrentTab = "Finder"
local Joining = false
local Running = true
local ScanCount = 0

local function setStatus(text)
    if statusLabel then
        statusLabel.Text = "● " .. text
    end
end

--==================================================
-- UTILITÁRIOS
--==================================================

local suffix = {
    K = 1e3,
    M = 1e6,
    B = 1e9,
    T = 1e12,
}

local function parseValue(value)
    local str = tostring(value):upper()
        :gsub(",", "")
        :gsub("%$", "")
        :gsub("/S", "")
        :gsub("/SEC", "")

    local num, suf = str:match("([%d%.]+)%s*([KMBT]?)")

    if not num then return nil end

    return tonumber(num) * (suffix[suf] or 1)
end

local function formatValue(n)
    if n >= 1e12 then
        return string.format("%.2fT", n / 1e12)
    elseif n >= 1e9 then
        return string.format("%.2fB", n / 1e9)
    elseif n >= 1e6 then
        return string.format("%.2fM", n / 1e6)
    elseif n >= 1e3 then
        return string.format("%.2fK", n / 1e3)
    end

    return tostring(math.floor(n))
end

local function getValue(model)
    for _, key in ipairs({
        "Income", "PerSecond", "Value",
        "Price", "Worth"
    }) do
        local attribute = model:GetAttribute(key)

        if attribute ~= nil then
            if type(attribute) == "number" then
                return attribute
            end

            local parsed = parseValue(attribute)
            if parsed then return parsed end
        end

        local child = model:FindFirstChild(key)

        if child and child:IsA("ValueBase") then
            if type(child.Value) == "number" then
                return child.Value
            end

            local parsed = parseValue(child.Value)
            if parsed then return parsed end
        end
    end

    for _, descendant in ipairs(model:GetDescendants()) do
        if descendant:IsA("TextLabel") then
            local parsed = parseValue(descendant.Text)

            if parsed then return parsed end
        end
    end
end

local function getRarity(model)
    local rarity = model:GetAttribute("Rarity")

    if rarity then return tostring(rarity) end

    local child = model:FindFirstChild("Rarity")

    if child and child:IsA("ValueBase") then
        return tostring(child.Value)
    end

    return "Unknown"
end

local function getMultiplier(model)
    local mult = model:GetAttribute("Multiplier")
        or model:GetAttribute("Mult")

    return tonumber(mult) or 1
end

local function isPet(model)
    if not model:IsA("Model") then
        return false
    end

    if Players:GetPlayerFromCharacter(model) then
        return false
    end

    if model:GetAttribute("IsPet")
        or model:GetAttribute("Pet") then
        return true
    end

    return model.Name:lower():find("pet") ~= nil
        or getValue(model) ~= nil
end

--==================================================
-- GUI
--==================================================

local gui = Instance.new("ScreenGui")
gui.Name = "DavidHubV2"
gui.ResetOnSpawn = false
gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local parentGui

pcall(function()
    if gethui then
        parentGui = gethui()
    end
end)

if not parentGui then
    parentGui = LP:WaitForChild("PlayerGui")
end

gui.Parent = parentGui

local scale = Instance.new("UIScale")
scale.Scale = Config.UIScale
scale.Parent = gui

local function corner(object, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 8)
    c.Parent = object
end

local function outline(object, color, thickness)
    local s = Instance.new("UIStroke")
    s.Color = color or COLORS.Green
    s.Thickness = thickness or 1
    s.Parent = object
end

local function createLabel(parent, text, size, color)
    local label = Instance.new("TextLabel")
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextSize = size or 14
    label.TextColor3 = color or COLORS.Text
    label.Font = Enum.Font.GothamBold
    label.Parent = parent

    return label
end

local function createButton(parent, text, size, position)
    local button = Instance.new("TextButton")
    button.Size = size
    button.Position = position
    button.BackgroundColor3 = COLORS.Card
    button.Text = text
    button.TextColor3 = COLORS.Text
    button.Font = Enum.Font.GothamBold
    button.TextSize = 13
    button.BorderSizePixel = 0
    button.AutoButtonColor = true
    button.Parent = parent

    corner(button, 8)
    outline(button, COLORS.GreenDark, 1)

    return button
end

--==================================================
-- JANELA PRINCIPAL
--==================================================

local main = Instance.new("Frame")
main.Name = "Main"
main.Size = UDim2.fromOffset(520, 560)
main.Position = UDim2.new(0.5, -260, 0.5, -280)
main.BackgroundColor3 = COLORS.Background
main.BorderSizePixel = 0
main.ClipsDescendants = true
main.Parent = gui

corner(main, 12)
outline(main, COLORS.Green, 2)

--==================================================
-- HEADER
--==================================================

local header = Instance.new("Frame")
header.Size = UDim2.new(1, 0, 0, 58)
header.BackgroundColor3 = COLORS.GreenDark
header.BorderSizePixel = 0
header.Parent = main

corner(header, 12)

local gradient = Instance.new("UIGradient")
gradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(55, 230, 125)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(10, 100, 55))
})
gradient.Rotation = 0
gradient.Parent = header

local title = createLabel(
    header,
    "DAVID HUB",
    22,
    Color3.new(1, 1, 1)
)

title.Position = UDim2.fromOffset(18, 4)
title.Size = UDim2.fromOffset(230, 30)
title.TextXAlignment = Enum.TextXAlignment.Left

local version = createLabel(
    header,
    "GREEN EDITION  •  V2",
    10,
    Color3.fromRGB(215, 255, 225)
)

version.Position = UDim2.fromOffset(20, 32)
version.Size = UDim2.fromOffset(180, 16)
version.TextXAlignment = Enum.TextXAlignment.Left

local minimize = createButton(
    header,
    "—",
    UDim2.fromOffset(36, 34),
    UDim2.new(1, -48, 0, 12)
)

minimize.TextSize = 20

local contentVisible = true

--==================================================
-- DRAG
--==================================================

do
    local dragging = false
    local dragStart
    local startPosition

    header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            dragStart = input.Position
            startPosition = main.Position
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if dragging and (
            input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch
        ) then
            local delta = input.Position - dragStart

            main.Position = UDim2.new(
                startPosition.X.Scale,
                startPosition.X.Offset + delta.X,
                startPosition.Y.Scale,
                startPosition.Y.Offset + delta.Y
            )
        end
    end)

    UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

--==================================================
-- SIDEBAR
--==================================================

local sidebar = Instance.new("Frame")
sidebar.Position = UDim2.fromOffset(10, 70)
sidebar.Size = UDim2.fromOffset(115, 475)
sidebar.BackgroundColor3 = COLORS.Panel
sidebar.BorderSizePixel = 0
sidebar.Parent = main

corner(sidebar, 9)

local finderTab = createButton(
    sidebar,
    "⌕  FINDER",
    UDim2.new(1, -12, 0, 42),
    UDim2.fromOffset(6, 10)
)

local settingsTab = createButton(
    sidebar,
    "⚙  SETTINGS",
    UDim2.new(1, -12, 0, 42),
    UDim2.fromOffset(6, 60)
)

local account = createLabel(
    sidebar,
    LP.DisplayName,
    11,
    COLORS.Muted
)

account.Position = UDim2.new(0, 5, 1, -35)
account.Size = UDim2.new(1, -10, 0, 20)
account.TextTruncate = Enum.TextTruncate.AtEnd

--==================================================
-- CONTENT
--==================================================

local content = Instance.new("Frame")
content.Position = UDim2.fromOffset(135, 70)
content.Size = UDim2.new(1, -145, 1, -85)
content.BackgroundTransparency = 1
content.Parent = main

--==================================================
-- FINDER PAGE
--==================================================

local finderPage = Instance.new("Frame")
finderPage.Size = UDim2.fromScale(1, 1)
finderPage.BackgroundTransparency = 1
finderPage.Parent = content

local finderTitle = createLabel(
    finderPage,
    "Egg Finder",
    20,
    COLORS.Green
)

finderTitle.Size = UDim2.new(1, 0, 0, 30)
finderTitle.TextXAlignment = Enum.TextXAlignment.Left

local description = createLabel(
    finderPage,
    "Encontre modelos disponíveis neste servidor.",
    10,
    COLORS.Muted
)

description.Position = UDim2.fromOffset(0, 30)
description.Size = UDim2.new(1, 0, 0, 20)
description.TextXAlignment = Enum.TextXAlignment.Left

-- STATUS

local statusBox = Instance.new("Frame")
statusBox.Position = UDim2.fromOffset(0, 55)
statusBox.Size = UDim2.new(1, 0, 0, 35)
statusBox.BackgroundColor3 = COLORS.Panel
statusBox.Parent = finderPage

corner(statusBox, 7)

local statusLabel = createLabel(
    statusBox,
    "● Aguardando escaneamento",
    11,
    COLORS.Green
)

statusLabel.Position = UDim2.fromOffset(10, 0)
statusLabel.Size = UDim2.new(1, -20, 1, 0)
statusLabel.TextXAlignment = Enum.TextXAlignment.Left

-- ESTATÍSTICAS

local stats = Instance.new("Frame")
stats.Position = UDim2.fromOffset(0, 100)
stats.Size = UDim2.new(1, 0, 0, 55)
stats.BackgroundTransparency = 1
stats.Parent = finderPage

local statsLayout = Instance.new("UIListLayout")
statsLayout.FillDirection = Enum.FillDirection.Horizontal
statsLayout.Padding = UDim.new(0, 6)
statsLayout.Parent = stats

local function statCard(name, value)
    local card = Instance.new("Frame")
    card.Size = UDim2.new(0.333, -5, 1, 0)
    card.BackgroundColor3 = COLORS.Panel
    card.Parent = stats

    corner(card, 7)

    local top = createLabel(card, name, 10, COLORS.Muted)
    top.Size = UDim2.new(1, 0, 0, 20)
    top.Position = UDim2.fromOffset(0, 5)

    local bottom = createLabel(card, value, 16, COLORS.Green)
    bottom.Size = UDim2.new(1, 0, 0, 25)
    bottom.Position = UDim2.fromOffset(0, 25)

    return bottom
end

local foundStat = statCard("FOUND", "0")
local serverStat = statCard("SERVER", "LOCAL")
local resultStat = statCard("RESULTS", "0")

-- CONTROLES

local controls = Instance.new("Frame")
controls.Position = UDim2.fromOffset(0, 165)
controls.Size = UDim2.new(1, 0, 0, 38)
controls.BackgroundTransparency = 1
controls.Parent = finderPage

local scanButton = createButton(
    controls,
    "SCAN NOW",
    UDim2.new(0.49, -3, 1, 0),
    UDim2.fromOffset(0, 0)
)

scanButton.BackgroundColor3 = COLORS.GreenDark

local autoButton = createButton(
    controls,
    "AUTO OFF",
    UDim2.new(0.49, -3, 1, 0),
    UDim2.new(0.51, 3, 0, 0)
)

-- LISTA

local listTitle = createLabel(
    finderPage,
    "RECENT RESULTS",
    12,
    COLORS.Green
)

listTitle.Position = UDim2.fromOffset(0, 212)
listTitle.Size = UDim2.new(1, 0, 0, 22)
listTitle.TextXAlignment = Enum.TextXAlignment.Left

local list = Instance.new("ScrollingFrame")
list.Position = UDim2.fromOffset(0, 238)
list.Size = UDim2.new(1, 0, 1, -240)
list.BackgroundColor3 = COLORS.Panel
list.BorderSizePixel = 0
list.ScrollBarThickness = 4
list.ScrollBarImageColor3 = COLORS.Green
list.AutomaticCanvasSize = Enum.AutomaticSize.Y
list.CanvasSize = UDim2.new()
list.Parent = finderPage

corner(list, 8)

local listLayout = Instance.new("UIListLayout")
listLayout.Padding = UDim.new(0, 6)
listLayout.SortOrder = Enum.SortOrder.LayoutOrder
listLayout.Parent = list

local listPadding = Instance.new("UIPadding")
listPadding.PaddingTop = UDim.new(0, 7)
listPadding.PaddingLeft = UDim.new(0, 7)
listPadding.PaddingRight = UDim.new(0, 7)
listPadding.Parent = list

--==================================================
-- SETTINGS PAGE
--==================================================

local settingsPage = Instance.new("ScrollingFrame")
settingsPage.Size = UDim2.fromScale(1, 1)
settingsPage.BackgroundTransparency = 1
settingsPage.BorderSizePixel = 0
settingsPage.ScrollBarThickness = 4
settingsPage.ScrollBarImageColor3 = COLORS.Green
settingsPage.AutomaticCanvasSize = Enum.AutomaticSize.Y
settingsPage.CanvasSize = UDim2.new()
settingsPage.Visible = false
settingsPage.Parent = content

local settingsLayout = Instance.new("UIListLayout")
settingsLayout.Padding = UDim.new(0, 10)
settingsLayout.Parent = settingsPage

local settingsPadding = Instance.new("UIPadding")
settingsPadding.PaddingBottom = UDim.new(0, 10)
settingsPadding.Parent = settingsPage

local settingsTitle = createLabel(
    settingsPage,
    "Advanced Settings",
    20,
    COLORS.Green
)

settingsTitle.Size = UDim2.new(1, 0, 0, 32)
settingsTitle.TextXAlignment = Enum.TextXAlignment.Left

local function settingRow(name, initial, callback)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1, -5, 0, 58)
    row.BackgroundColor3 = COLORS.Panel
    row.Parent = settingsPage

    corner(row, 8)

    local nameLabel = createLabel(
        row,
        name,
        12,
        COLORS.Text
    )

    nameLabel.Position = UDim2.fromOffset(10, 5)
    nameLabel.Size = UDim2.new(1, -105, 1, -10)
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left

    local input = Instance.new("TextBox")
    input.Position = UDim2.new(1, -90, 0.5, -15)
    input.Size = UDim2.fromOffset(78, 30)
    input.BackgroundColor3 = COLORS.Background
    input.TextColor3 = COLORS.Green
    input.Text = tostring(initial)
    input.Font = Enum.Font.GothamBold
    input.TextSize = 12
    input.ClearTextOnFocus = false
    input.BorderSizePixel = 0
    input.Parent = row

    corner(input, 6)

    input.FocusLost:Connect(function()
        callback(input)
    end)

    return input
end

settingRow("Minimum value (B/s)", Config.MinB, function(input)
    local value = tonumber(input.Text)
    if not value then value = Config.MinB end

    Config.MinB = math.clamp(value, 0.01, 100000)
    input.Text = tostring(Config.MinB)

    if Config.MaxB < Config.MinB then
        Config.MaxB = Config.MinB
    end
end)

settingRow("Maximum value (B/s)", Config.MaxB, function(input)
    local value = tonumber(input.Text)
    if not value then value = Config.MaxB end

    Config.MaxB = math.max(value, Config.MinB)
    input.Text = tostring(Config.MaxB)
end)

settingRow("UI Scale (0.6 - 1.4)", Config.UIScale, function(input)
    local value = tonumber(input.Text) or 1

    Config.UIScale = math.clamp(value, 0.6, 1.4)
    scale.Scale = Config.UIScale
    input.Text = tostring(Config.UIScale)
end)

settingRow("Scan interval (3 - 30s)", Config.ScanInterval, function(input)
    local value = tonumber(input.Text) or 5

    Config.ScanInterval = math.clamp(value, 3, 30)
    input.Text = tostring(Config.ScanInterval)
end)

settingRow("Force Join delay (2 - 5s)", Config.Delay, function(input)
    local value = tonumber(input.Text) or 2

    Config.Delay = math.clamp(value, 2, 5)
    input.Text = tostring(Config.Delay)
end)

-- RARIDADE

local rarityRow = Instance.new("Frame")
rarityRow.Size = UDim2.new(1, -5, 0, 58)
rarityRow.BackgroundColor3 = COLORS.Panel
rarityRow.Parent = settingsPage

corner(rarityRow, 8)

local rarityLabel = createLabel(
    rarityRow,
    "Rarity Filter",
    12,
    COLORS.Text
)

rarityLabel.Position = UDim2.fromOffset(10, 0)
rarityLabel.Size = UDim2.new(0.4, 0, 1, 0)
rarityLabel.TextXAlignment = Enum.TextXAlignment.Left

local rarityButton = createButton(
    rarityRow,
    Config.Rarity,
    UDim2.fromOffset(115, 34),
    UDim2.new(1, -125, 0.5, -17)
)

rarityButton.MouseButton1Click:Connect(function()
    local index = table.find(RARITIES, Config.Rarity) or 1
    Config.Rarity = RARITIES[index % #RARITIES + 1]
    rarityButton.Text = Config.Rarity
end)

--==================================================
-- NAVEGAÇÃO
--==================================================

local function changeTab(tab)
    CurrentTab = tab

    finderPage.Visible = tab == "Finder"
    settingsPage.Visible = tab == "Settings"

    finderTab.BackgroundColor3 =
        tab == "Finder" and COLORS.GreenDark or COLORS.Card

    settingsTab.BackgroundColor3 =
        tab == "Settings" and COLORS.GreenDark or COLORS.Card
end

finderTab.MouseButton1Click:Connect(function()
    changeTab("Finder")
end)

settingsTab.MouseButton1Click:Connect(function()
    changeTab("Settings")
end)

minimize.MouseButton1Click:Connect(function()
    contentVisible = not contentVisible
    sidebar.Visible = contentVisible
    content.Visible = contentVisible

    main.Size = contentVisible
        and UDim2.fromOffset(520, 560)
        or UDim2.fromOffset(520, 58)
end)

--==================================================
-- RESULTADOS
--==================================================

local function ago(timestamp)
    local difference = math.max(0, os.time() - timestamp)

    if difference < 60 then
        return difference .. "s ago"
    end

    return math.floor(difference / 60) .. "m ago"
end

local function updateStats()
    foundStat.Text = tostring(ScanCount)
    resultStat.Text = tostring(#Entries)
end

local function clearOldest()
    if #Entries >= Config.MaxResults then
        local oldest = table.remove(Entries, 1)

        if oldest.frame then
            oldest.frame:Destroy()
        end
    end
end

local function addEntry(name, value, rarity, mult, jobId)
    clearOldest()

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -4, 0, 72)
    frame.BackgroundColor3 = COLORS.Card
    frame.BorderSizePixel = 0
    frame.Parent = list

    corner(frame, 8)
    outline(frame, COLORS.GreenDark, 1)

    local nameLabel = createLabel(
        frame,
        name,
        13,
        RARITY_COLOR[rarity] or COLORS.Text
    )

    nameLabel.Position = UDim2.fromOffset(12, 7)
    nameLabel.Size = UDim2.new(1, -100, 0, 20)
    nameLabel.TextXAlignment = Enum.Text
