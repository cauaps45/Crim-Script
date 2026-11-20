--[[
    NEXUS UI | CRIMINALITY (VERSÃO EXECUTOR COMPLETA)
    - Ícone: Coelho estilizado em alto contraste (P&B) via UI nativa
    - Sem botão "OCULTAR UI": botão "—" na barra de título faz a ocultação e "FECHAR E LIMPAR" ocupa a maior parte da faixa
    - Crates & Ground Items: Highlight na malha real + supressão de Highlight nativo perto
    - Mystery Boxes: Workspace.Map.MysteryBoxes (ex.: HALLOWS_MysteryBox - mais próxima)
    - Registers / Safes / ATMs / Dealers / Trash / Night Vision
    - Player ESP: Filtro por nomes separados por vírgula ("uraraka, KFC, sincryz")
]]

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Lighting = game:GetService("Lighting")

local Settings = {
    Players = {
        Enabled = false,
        Color = Color3.fromRGB(255, 255, 255),
        Distance = 250,
        ShowName = true,
        ShowHealth = true,
        ShowWeapon = true,
        ShowDist = true,
        FilterEnabled = false,
        FilterRaw = ""
    },
    MysteryBoxes= { Enabled = false, Color = Color3.fromRGB(16, 185, 129), Distance = 500, ShowDist = true },
    Registers   = { Enabled = false, ColorClosed = Color3.fromRGB(34, 197, 94), ColorOpen = Color3.fromRGB(239, 68, 68), Distance = 250, ShowDist = true },
    Safes       = { Enabled = false, ColorClosed = Color3.fromRGB(34, 197, 94), ColorOpen = Color3.fromRGB(239, 68, 68), Distance = 250, ShowDist = true },
    ATMs        = { Enabled = false, Color = Color3.fromRGB(170, 0, 255), Distance = 250, ShowDist = true },
    Dealers     = { Enabled = false, Color = Color3.fromRGB(0, 150, 255), Distance = 250, ShowDist = true },
    Crates      = { Enabled = false, Color = Color3.fromRGB(255, 120, 0), Distance = 250, ShowDist = false },
    GroundItems = { Enabled = false, Color = Color3.fromRGB(255, 230, 0), CashColor = Color3.fromRGB(74, 222, 128), Distance = 250, ShowDist = true },
    Trash       = { Enabled = false, Color = Color3.fromRGB(139, 69, 19), Distance = 250, ShowDist = true },
    NightVision = false,
    ToggleKeyName = ""
}

local activeToggleKey = nil
local listeningForKey = false

local FULL_SIZE = UDim2.new(0, 425, 0, 565)

local OriginalLighting = {
    Ambient = Lighting.Ambient,
    OutdoorAmbient = Lighting.OutdoorAmbient,
    Brightness = Lighting.Brightness,
    GlobalShadows = Lighting.GlobalShadows,
    FogEnd = Lighting.FogEnd
}

local function ensureNVGEffect()
    local cc = Lighting:FindFirstChild("NexusNVG")
    if not cc or not cc:IsA("ColorCorrectionEffect") then
        cc = Instance.new("ColorCorrectionEffect")
        cc.Name = "NexusNVG"
        cc.Parent = Lighting
    end
    cc.TintColor = Color3.fromRGB(115, 255, 135)
    cc.Contrast = 0.25
    cc.Saturation = -0.35
    cc.Brightness = 0.12
    cc.Enabled = Settings.NightVision
    return cc
end

local nvgCorrection = ensureNVGEffect()

local function applyNightVisionFrame()
    nvgCorrection = ensureNVGEffect()
    if Settings.NightVision then
        Lighting.Ambient = Color3.fromRGB(95, 215, 110)
        Lighting.OutdoorAmbient = Color3.fromRGB(95, 215, 110)
        Lighting.Brightness = 2.4
        Lighting.GlobalShadows = false
        Lighting.FogEnd = 100000
        nvgCorrection.Enabled = true
    else
        nvgCorrection.Enabled = false
    end
end

local function setNightVision(enabled)
    local wasEnabled = Settings.NightVision
    Settings.NightVision = enabled
    if enabled and not wasEnabled then
        OriginalLighting.Ambient = Lighting.Ambient
        OriginalLighting.OutdoorAmbient = Lighting.OutdoorAmbient
        OriginalLighting.Brightness = Lighting.Brightness
        OriginalLighting.GlobalShadows = Lighting.GlobalShadows
        OriginalLighting.FogEnd = Lighting.FogEnd
        applyNightVisionFrame()
    elseif not enabled then
        Lighting.Ambient = OriginalLighting.Ambient
        Lighting.OutdoorAmbient = OriginalLighting.OutdoorAmbient
        Lighting.Brightness = OriginalLighting.Brightness
        Lighting.GlobalShadows = OriginalLighting.GlobalShadows
        Lighting.FogEnd = OriginalLighting.FogEnd
        if nvgCorrection and nvgCorrection.Parent then
            nvgCorrection.Enabled = false
        end
    end
end

local function formatKeyLabel(kc)
    if not kc then return "nenhuma" end
    if kc == Enum.KeyCode.RightAlt then
        return "AltGr (RightAlt)"
    end
    return kc.Name
end

local function colorsMatch(c1, c2)
    if not c1 or not c2 then return false end
    return math.abs(c1.R - c2.R) < 0.02 and math.abs(c1.G - c2.G) < 0.02 and math.abs(c1.B - c2.B) < 0.02
end

local function computeScrapColor()
    local brown = Color3.fromRGB(139, 69, 19)
    local fallback = Color3.fromRGB(56, 189, 248)
    local usedColors = {
        Settings.Players.Color,
        Settings.MysteryBoxes.Color,
        Settings.Registers.ColorClosed,
        Settings.Registers.ColorOpen,
        Settings.Safes.ColorClosed,
        Settings.Safes.ColorOpen,
        Settings.ATMs.Color,
        Settings.Dealers.Color,
        Settings.Crates.Color,
        Settings.GroundItems.Color,
        Settings.GroundItems.CashColor
    }

    for i = 1, #usedColors do
        if colorsMatch(brown, usedColors[i]) then
            return fallback
        end
    end
    return brown
end

Settings.Trash.Color = computeScrapColor()

local parsedTargets = {}
local function refreshTargetList(raw)
    table.clear(parsedTargets)
    if not raw or raw:match("^%s*$") then return end
    for token in raw:gmatch("[^,]+") do
        local clean = token:match("^%s*(.-)%s*$"):lower()
        if clean ~= "" then
            table.insert(parsedTargets, clean)
        end
    end
end

local function isPlayerAllowedByFilter(plr)
    if not Settings.Players.FilterEnabled then
        return true
    end
    if #parsedTargets == 0 then
        return true
    end
    local pName = plr.Name:lower()
    local dName = plr.DisplayName and plr.DisplayName:lower() or ""
    for i = 1, #parsedTargets do
        local target = parsedTargets[i]
        if pName:find(target, 1, true) or (dName ~= "" and dName:find(target, 1, true)) then
            return true
        end
    end
    return false
end

local function buildRabbitIcon(parentFrame)
    local holder = Instance.new("Frame")
    holder.Name = "RabbitIcon"
    holder.Size = UDim2.new(1, 0, 1, 0)
    holder.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
    holder.BorderSizePixel = 0
    holder.ClipsDescendants = true
    holder.Parent = parentFrame
    Instance.new("UICorner", holder).CornerRadius = UDim.new(0, 7)

    local white = Color3.fromRGB(248, 250, 252)
    local black = Color3.fromRGB(10, 10, 14)

    local earL = Instance.new("Frame")
    earL.Size = UDim2.new(0.22, 0, 0.44, 0)
    earL.Position = UDim2.new(0.22, 0, 0.04, 0)
    earL.Rotation = -22
    earL.BackgroundColor3 = white
    earL.BorderSizePixel = 0
    earL.Parent = holder
    Instance.new("UICorner", earL).CornerRadius = UDim.new(1, 0)

    local earLInner = Instance.new("Frame")
    earLInner.Size = UDim2.new(0.45, 0, 0.65, 0)
    earLInner.Position = UDim2.new(0.28, 0, 0.16, 0)
    earLInner.BackgroundColor3 = black
    earLInner.BorderSizePixel = 0
    earLInner.Parent = earL
    Instance.new("UICorner", earLInner).CornerRadius = UDim.new(1, 0)

    local earR = Instance.new("Frame")
    earR.Size = UDim2.new(0.22, 0, 0.44, 0)
    earR.Position = UDim2.new(0.56, 0, 0.04, 0)
    earR.Rotation = 22
    earR.BackgroundColor3 = white
    earR.BorderSizePixel = 0
    earR.Parent = holder
    Instance.new("UICorner", earR).CornerRadius = UDim.new(1, 0)

    local earRInner = Instance.new("Frame")
    earRInner.Size = UDim2.new(0.45, 0, 0.65, 0)
    earRInner.Position = UDim2.new(0.28, 0, 0.16, 0)
    earRInner.BackgroundColor3 = black
    earRInner.BorderSizePixel = 0
    earRInner.Parent = earR
    Instance.new("UICorner", earRInner).CornerRadius = UDim.new(1, 0)

    local chest = Instance.new("Frame")
    chest.Size = UDim2.new(0.44, 0, 0.34, 0)
    chest.Position = UDim2.new(0.28, 0, 0.58, 0)
    chest.Rotation = 45
    chest.BackgroundColor3 = white
    chest.BorderSizePixel = 0
    chest.Parent = holder
    Instance.new("UICorner", chest).CornerRadius = UDim.new(0, 5)

    local head = Instance.new("Frame")
    head.Size = UDim2.new(0.44, 0, 0.38, 0)
    head.Position = UDim2.new(0.28, 0, 0.32, 0)
    head.BackgroundColor3 = white
    head.BorderSizePixel = 0
    head.Parent = holder
    Instance.new("UICorner", head).CornerRadius = UDim.new(1, 0)

    local muzzle = Instance.new("Frame")
    muzzle.Size = UDim2.new(0.54, 0, 0.28, 0)
    muzzle.Position = UDim2.new(0.23, 0, 0.46, 0)
    muzzle.BackgroundColor3 = white
    muzzle.BorderSizePixel = 0
    muzzle.Parent = holder
    Instance.new("UICorner", muzzle).CornerRadius = UDim.new(1, 0)

    local muzzleStroke = Instance.new("UIStroke")
    muzzleStroke.Color = black
    muzzleStroke.Thickness = 1.4
    muzzleStroke.Parent = muzzle

    local eyeL = Instance.new("Frame")
    eyeL.Size = UDim2.new(0.11, 0, 0.12, 0)
    eyeL.Position = UDim2.new(0.32, 0, 0.41, 0)
    eyeL.BackgroundColor3 = black
    eyeL.BorderSizePixel = 0
    eyeL.Parent = holder
    Instance.new("UICorner", eyeL).CornerRadius = UDim.new(1, 0)

    local eyeR = Instance.new("Frame")
    eyeR.Size = UDim2.new(0.11, 0, 0.12, 0)
    eyeR.Position = UDim2.new(0.57, 0, 0.41, 0)
    eyeR.BackgroundColor3 = black
    eyeR.BorderSizePixel = 0
    eyeR.Parent = holder
    Instance.new("UICorner", eyeR).CornerRadius = UDim.new(1, 0)

    local nose = Instance.new("Frame")
    nose.Size = UDim2.new(0.12, 0, 0.09, 0)
    nose.Position = UDim2.new(0.44, 0, 0.56, 0)
    nose.Rotation = 45
    nose.BackgroundColor3 = black
    nose.BorderSizePixel = 0
    nose.Parent = holder
    Instance.new("UICorner", nose).CornerRadius = UDim.new(0, 2)
end

local GUI_NAME = "NexusUI_Criminality"
if CoreGui:FindFirstChild(GUI_NAME) then
    CoreGui[GUI_NAME]:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = GUI_NAME
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui

local PriorityHighlightFolder = Instance.new("Folder")
PriorityHighlightFolder.Name = "NexusActiveHighlights"
PriorityHighlightFolder.Parent = ScreenGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = FULL_SIZE
MainFrame.Position = UDim2.new(0.04, 0, 0.07, 0)
MainFrame.BackgroundColor3 = Color3.fromRGB(16, 15, 24)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.ClipsDescendants = true
MainFrame.Parent = ScreenGui
Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)

local Stroke = Instance.new("UIStroke")
Stroke.Color = Color3.fromRGB(147, 51, 234)
Stroke.Thickness = 1.4
Stroke.Parent = MainFrame

local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 48)
TitleBar.BackgroundColor3 = Color3.fromRGB(26, 23, 38)
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame
Instance.new("UICorner", TitleBar).CornerRadius = UDim.new(0, 12)

local IconBadge = Instance.new("Frame")
IconBadge.Size = UDim2.new(0, 34, 0, 34)
IconBadge.Position = UDim2.new(0, 9, 0.5, -17)
IconBadge.BackgroundColor3 = Color3.fromRGB(10, 10, 14)
IconBadge.BorderSizePixel = 0
IconBadge.Parent = TitleBar
Instance.new("UICorner", IconBadge).CornerRadius = UDim.new(0, 8)

local IconStroke = Instance.new("UIStroke")
IconStroke.Color = Color3.fromRGB(192, 132, 252)
IconStroke.Thickness = 1.2
IconStroke.Parent = IconBadge

buildRabbitIcon(IconBadge)

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -98, 1, 0)
Title.Position = UDim2.new(0, 52, 0, 0)
Title.BackgroundTransparency = 1
Title.TextColor3 = Color3.fromRGB(226, 196, 255)
Title.Text = "Nexus UI | Criminality"
Title.TextSize = 14
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TitleBar

local MinBtn = Instance.new("TextButton")
MinBtn.Size = UDim2.new(0, 30, 0, 30)
MinBtn.Position = UDim2.new(1, -40, 0.5, -15)
MinBtn.BackgroundColor3 = Color3.fromRGB(45, 34, 66)
MinBtn.TextColor3 = Color3.fromRGB(233, 213, 255)
MinBtn.Text = "-"
MinBtn.TextSize = 16
MinBtn.Font = Enum.Font.GothamBold
MinBtn.Parent = TitleBar
Instance.new("UICorner", MinBtn).CornerRadius = UDim.new(0, 7)

local BodyContainer = Instance.new("Frame")
BodyContainer.Size = UDim2.new(1, 0, 1, -48)
BodyContainer.Position = UDim2.new(0, 0, 0, 48)
BodyContainer.BackgroundTransparency = 1
BodyContainer.Visible = true
BodyContainer.Parent = MainFrame

local ReopenBtn = Instance.new("Frame")
ReopenBtn.Size = UDim2.new(0, 102, 0, 34)
ReopenBtn.Position = UDim2.new(0.04, 0, 0.07, 0)
ReopenBtn.BackgroundColor3 = Color3.fromRGB(27, 24, 38)
ReopenBtn.Visible = false
ReopenBtn.Active = true
ReopenBtn.Parent = ScreenGui
Instance.new("UICorner", ReopenBtn).CornerRadius = UDim.new(0, 8)

local ReopenStroke = Instance.new("UIStroke")
ReopenStroke.Color = Color3.fromRGB(147, 51, 234)
ReopenStroke.Thickness = 1.2
ReopenStroke.Parent = ReopenBtn

local ReopenIconWrap = Instance.new("Frame")
ReopenIconWrap.Size = UDim2.new(0, 26, 0, 26)
ReopenIconWrap.Position = UDim2.new(0, 4, 0.5, -13)
ReopenIconWrap.BackgroundTransparency = 1
ReopenIconWrap.Parent = ReopenBtn
buildRabbitIcon(ReopenIconWrap)

local ReopenLbl = Instance.new("TextLabel")
ReopenLbl.Size = UDim2.new(1, -34, 1, 0)
ReopenLbl.Position = UDim2.new(0, 34, 0, 0)
ReopenLbl.BackgroundTransparency = 1
ReopenLbl.Text = "NEXUS"
ReopenLbl.TextColor3 = Color3.fromRGB(226, 196, 255)
ReopenLbl.TextSize = 11
ReopenLbl.Font = Enum.Font.GothamBold
ReopenLbl.TextXAlignment = Enum.TextXAlignment.Left
ReopenLbl.Parent = ReopenBtn

local function setUIVisible(vis)
    listeningForKey = false
    MainFrame.Visible = vis
    ReopenBtn.Visible = not vis
end

MinBtn.MouseButton1Click:Connect(function()
    setUIVisible(false)
end)

do
    local dragging = false
    local dragMoved = false
    local dragStartPos = Vector2.zero
    local frameStartPos = UDim2.new()

    ReopenBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragMoved = false
            dragStartPos = input.Position
            frameStartPos = ReopenBtn.Position
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStartPos
            if delta.Magnitude > 4 then
                dragMoved = true
            end
            ReopenBtn.Position = UDim2.new(
                frameStartPos.X.Scale,
                frameStartPos.X.Offset + delta.X,
                frameStartPos.Y.Scale,
                frameStartPos.Y.Offset + delta.Y
            )
        end
    end)

    ReopenBtn.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            if dragging and not dragMoved then
                setUIVisible(true)
            end
            dragging = false
            dragMoved = false
        end
    end)
end

local ExitBtn = Instance.new("TextButton")
ExitBtn.Size = UDim2.new(0.92, 0, 0, 32)
ExitBtn.Position = UDim2.new(0.04, 0, 0, 8)
ExitBtn.BackgroundColor3 = Color3.fromRGB(126, 34, 206)
ExitBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
ExitBtn.Text = "FECHAR E LIMPAR"
ExitBtn.TextSize = 11
ExitBtn.Font = Enum.Font.GothamBold
ExitBtn.Parent = BodyContainer
Instance.new("UICorner", ExitBtn).CornerRadius = UDim.new(0, 7)

local Scroll = Instance.new("ScrollingFrame")
Scroll.Size = UDim2.new(1, 0, 1, -48)
Scroll.Position = UDim2.new(0, 0, 0, 46)
Scroll.BackgroundTransparency = 1
Scroll.ScrollBarThickness = 6
Scroll.ScrollBarImageColor3 = Color3.fromRGB(168, 85, 247)
Scroll.Parent = BodyContainer

local yPos = 8
local function addSection(txt)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(0.92, 0, 0, 24)
    l.Position = UDim2.new(0.04, 0, 0, yPos)
    l.BackgroundTransparency = 1
    l.Text = string.upper(txt)
    l.TextColor3 = Color3.fromRGB(192, 132, 252)
    l.TextSize = 11
    l.Font = Enum.Font.GothamBold
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = Scroll
    yPos += 26
end

addSection("Interface")
local BindBtn = Instance.new("TextButton")
BindBtn.Size = UDim2.new(0.92, 0, 0,
