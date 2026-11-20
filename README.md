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
    RebelDealers= { Enabled = false, Color = Color3.fromRGB(255, 0, 0), Distance = 250, ShowDist = true },
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
        Settings.RebelDealers.Color,
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

-- Botão "-" no canto superior direito com a mesma função de ocultar/reabrir
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

-- Botão "FECHAR E LIMPAR" maior ocupando a maior parte do espaço sem encostar nas bordas
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
BindBtn.Size = UDim2.new(0.92, 0, 0, 30)
BindBtn.Position = UDim2.new(0.04, 0, 0, yPos)
BindBtn.BackgroundColor3 = Color3.fromRGB(34, 30, 46)
BindBtn.TextColor3 = Color3.fromRGB(233, 213, 255)
BindBtn.Text = "DEFINIR TECLA DE OCULTAR (CLIQUE AQUI)"
BindBtn.TextSize = 11
BindBtn.Font = Enum.Font.GothamSemibold
BindBtn.Parent = Scroll
Instance.new("UICorner", BindBtn).CornerRadius = UDim.new(0, 7)
yPos += 36

local KeyStatus = Instance.new("TextLabel")
KeyStatus.Size = UDim2.new(0.92, 0, 0, 18)
KeyStatus.Position = UDim2.new(0.04, 0, 0, yPos)
KeyStatus.BackgroundTransparency = 1
KeyStatus.Text = "Tecla atual: nenhuma (clique acima e aperte a tecla)"
KeyStatus.TextColor3 = Color3.fromRGB(160, 155, 180)
KeyStatus.TextSize = 10
KeyStatus.Font = Enum.Font.Gotham
KeyStatus.TextXAlignment = Enum.TextXAlignment.Left
KeyStatus.Parent = Scroll
yPos += 24

BindBtn.MouseButton1Click:Connect(function()
    listeningForKey = true
    BindBtn.Text = "PRESSIONE QUALQUER TECLA (ALT, ALTGR, ETC.)..."
    BindBtn.TextColor3 = Color3.fromRGB(253, 186, 116)
end)

local function destroyPlayerVisual(pChar)
    if not pChar then return end
    for _, d in ipairs(pChar:GetChildren()) do
        if d.Name == "NexusHL" or d.Name == "NexusPlrGui" then
            pcall(function() d:Destroy() end)
        end
    end
end

local function addToggle(name, getFn, setFn)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0.92, 0, 0, 32)
    b.Position = UDim2.new(0.04, 0, 0, yPos)
    b.BackgroundColor3 = Color3.fromRGB(28, 27, 36)
    b.TextSize = 12
    b.Font = Enum.Font.GothamSemibold
    b.Parent = Scroll
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 7)

    local function upd()
        local on = getFn()
        b.Text = name .. (on and " [ON]" or " [OFF]")
        b.TextColor3 = on and Color3.fromRGB(192, 132, 252) or Color3.fromRGB(160, 160, 175)
    end
    upd()
    b.MouseButton1Click:Connect(function()
        local nextVal = not getFn()
        setFn(nextVal)
        upd()
        if name == "Players ESP" and not nextVal then
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= LocalPlayer and plr.Character then
                    destroyPlayerVisual(plr.Character)
                end
            end
        end
    end)
    yPos += 36
end

local function addDistBox(name, cfg)
    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0.92, 0, 0, 28)
    box.Position = UDim2.new(0.04, 0, 0, yPos)
    box.BackgroundColor3 = Color3.fromRGB(24, 23, 31)
    box.TextColor3 = Color3.fromRGB(220, 215, 240)
    box.Text = name .. " Dist (1-2000): " .. tostring(cfg.Distance)
    box.TextSize = 11
    box.Font = Enum.Font.Gotham
    box.Parent = Scroll
    Instance.new("UICorner", box).CornerRadius = UDim.new(0, 6)

    box.FocusLost:Connect(function()
        local n = tonumber(box.Text:match("%d+"))
        if n then cfg.Distance = math.clamp(n, 1, 2000) end
        box.Text = name .. " Dist (1-2000): " .. tostring(cfg.Distance)
    end)
    yPos += 32
end

addSection("Visuals")
addToggle("Night Vision", function() return Settings.NightVision end, function(v) setNightVision(v) end)

addSection("Player ESP")
addToggle("Players ESP", function() return Settings.Players.Enabled end, function(v) Settings.Players.Enabled = v end)
addDistBox("Players", Settings.Players)
addToggle("Nome", function() return Settings.Players.ShowName end, function(v) Settings.Players.ShowName = v end)
addToggle("Vida", function() return Settings.Players.ShowHealth end, function(v) Settings.Players.ShowHealth = v end)
addToggle("Item Equipado", function() return Settings.Players.ShowWeapon end, function(v) Settings.Players.ShowWeapon = v end)
addToggle("Distância", function() return Settings.Players.ShowDist end, function(v) Settings.Players.ShowDist = v end)
addToggle("Apenas Alvos Específicos", function() return Settings.Players.FilterEnabled end, function(v) Settings.Players.FilterEnabled = v end)

local TargetBox = Instance.new("TextBox")
TargetBox.Size = UDim2.new(0.92, 0, 0, 30)
TargetBox.Position = UDim2.new(0.04, 0, 0, yPos)
TargetBox.BackgroundColor3 = Color3.fromRGB(24, 23, 31)
TargetBox.TextColor3 = Color3.fromRGB(233, 213, 255)
TargetBox.PlaceholderText = "Alvos (ex: uraraka, KFC, sincryz)"
TargetBox.PlaceholderColor3 = Color3.fromRGB(120, 115, 145)
TargetBox.Text = Settings.Players.FilterRaw
TargetBox.ClearTextOnFocus = false
TargetBox.TextSize = 11
TargetBox.Font = Enum.Font.Gotham
TargetBox.Parent = Scroll
Instance.new("UICorner", TargetBox).CornerRadius = UDim.new(0, 6)

TargetBox.FocusLost:Connect(function()
    Settings.Players.FilterRaw = TargetBox.Text
    refreshTargetList(Settings.Players.FilterRaw)
end)
yPos += 36

local function addObjBlock(label, cfg, noDistToggle)
    addSection(label)
    addToggle(label, function() return cfg.Enabled end, function(v) cfg.Enabled = v end)
    if not noDistToggle then
        addToggle("Exibir Distância", function() return cfg.ShowDist end, function(v) cfg.ShowDist = v end)
    end
    addDistBox(label, cfg)
end

addObjBlock("Mystery Boxes ESP", Settings.MysteryBoxes, false)
addObjBlock("Registers ESP", Settings.Registers, false)
addObjBlock("Safes ESP", Settings.Safes, false)
addObjBlock("ATMs ESP", Settings.ATMs, false)
addObjBlock("Dealers ESP", Settings.Dealers, false)
addObjBlock("Rebel Dealer ESP", Settings.RebelDealers, false)
addObjBlock("Crates ESP", Settings.Crates, true)
addObjBlock("Ground Items ESP", Settings.GroundItems, false)
addObjBlock("Trash ESP", Settings.Trash, false)

Scroll.CanvasSize = UDim2.new(0, 0, 0, yPos + 24)

local MetaCache = setmetatable({}, { __mode = "k" })

local function isCashName(str)
    if not str then return false end
    local l = str:lower()
    return l:find("cash", 1, true) ~= nil or l:find("bread", 1, true) ~= nil or l:find("money", 1, true) ~= nil
end

local function resolveAnchorPart(obj)
    if not obj then return nil end
    if obj:IsA("BasePart") then return obj end
    if obj:IsA("Model") then
        if obj.PrimaryPart and obj.PrimaryPart:IsA("BasePart") then
            return obj.PrimaryPart
        end
        for _, ch in ipairs(obj:GetChildren()) do
            if ch:IsA("BasePart") then
                return ch
            end
        end
        return obj:FindFirstChildWhichIsA("BasePart", true)
    elseif obj:IsA("Tool") or obj:IsA("Accessory") then
        local h = obj:FindFirstChild("Handle")
        if h and h:IsA("BasePart") then return h end
        for _, ch in ipairs(obj:GetChildren()) do
            if ch:IsA("BasePart") then return ch end
        end
    end
    return nil
end

local function getMeta(obj, kind)
    local m = MetaCache[obj]
    if m then return m end

    local anchor = resolveAnchorPart(obj)
    local low = obj.Name:lower()
    local isCash = isCashName(obj.Name) or (obj.Parent and isCashName(obj.Parent.Name))
    local label = ""
    local brokenVal = obj:FindFirstChild("Values") and obj.Values:FindFirstChild("Broken")
    local prompt = obj:FindFirstChildWhichIsA("ProximityPrompt", true)

    if kind == "mystery" then
        label = obj.Name
    elseif kind == "register" then
        label = "register"
    elseif kind == "safe" then
        label = low:find("medium", 1, true) and "medium safe" or "safe"
    elseif kind == "atm" or kind == "dealer" then
        label = obj.Name
    elseif kind == "ground" then
        label = isCash and "cash" or "item"
    elseif kind == "scrap" then
        label = "scrap"
    end

    m = {
        anchor = anchor,
        isCash = isCash,
        label = label,
        brokenVal = brokenVal,
        prompt = prompt
    }
    MetaCache[obj] = m
    return m
end

local Cache = {
    MysteryBoxes = {}, Registers = {}, Safes = {}, ATMs = {}, Dealers = {},
    RebelDealers = {}, Crates = {}, GroundItems = {}, Trash = {}
}

local VisualRegistry = {}
local activeHighlightBudget = 0
local MAX_TOTAL_HIGHLIGHTS = 45

local function isBrokenState(meta)
    if meta.brokenVal and typeof(meta.brokenVal.Value) == "boolean" then
        return meta.brokenVal.Value
    end
    if meta.prompt then
        return not meta.prompt.Enabled
    end
    return false
end

local function getStateColor(cfg, meta)
    return isBrokenState(meta) and cfg.ColorOpen or cfg.ColorClosed
end

local function suppressCompetingHighlights(target, keepHighlight)
    if not target then return end
    for _, desc in ipairs(target:GetDescendants()) do
        if desc:IsA("Highlight") and desc ~= keepHighlight then
            pcall(function()
                desc.Enabled = false
            end)
        end
    end
    local parentObj = target.Parent
    if parentObj and parentObj ~= Workspace then
        for _, child in ipairs(parentObj:GetChildren()) do
            if child:IsA("Highlight") and child ~= keepHighlight then
                if child.Adornee == target or child.Adornee == nil then
                    pcall(function()
                        child.Enabled = false
                    end)
                end
            end
        end
    end
end

local function ensureVisual(target, anchorPart, noTag, isPriorityHighlight)
    if not target or not target.Parent then return nil end
    local entry = VisualRegistry[target]
    local desiredParent = isPriorityHighlight and PriorityHighlightFolder or target

    if entry and entry.hl and entry.hl.Parent == desiredParent then
        if noTag or (entry.gui and entry.gui.Parent and entry.lbl and entry.lbl.Parent == entry.gui) then
            return entry
        end
    end

    if entry and entry.hl then
        pcall(function() entry.hl:Destroy() end)
    end

    local hl = Instance.new("Highlight")
    hl.Name = "NexusHL"
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.FillTransparency = isPriorityHighlight and 0.36 or 0.55
    hl.OutlineTransparency = isPriorityHighlight and 0.0 or 0.05
    hl.Adornee = target
    hl.Parent = desiredParent

    local gui, lbl = nil, nil
    if not noTag then
        local tagParent = anchorPart or target
        gui = tagParent:FindFirstChild("NexusTag")
        if not gui or not gui:IsA("BillboardGui") then
            gui = Instance.new("BillboardGui")
            gui.Name = "NexusTag"
            gui.Size = UDim2.new(0, 195, 0, 26)
            gui.StudsOffset = Vector3.new(0, 2.2, 0)
            gui.AlwaysOnTop = true
            gui.MaxDistance = math.huge
            gui.ClipsDescendants = false
            gui.Adornee = tagParent
            gui.Parent = tagParent
        end

        lbl = gui:FindFirstChild("Lbl")
        if not lbl or not lbl:IsA("TextLabel") then
            lbl = Instance.new("TextLabel")
            lbl.Name = "Lbl"
            lbl.Size = UDim2.new(1, 0, 1, 0)
            lbl.BackgroundTransparency = 1
            lbl.TextSize = 13
            lbl.Font = Enum.Font.GothamBold
            lbl.TextStrokeTransparency = 0.25
            lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
            lbl.ClipsDescendants = false
            lbl.Parent = gui
        end
    else
        if anchorPart and anchorPart:FindFirstChild("NexusTag") then
            anchorPart.NexusTag:Destroy()
        end
        if target:FindFirstChild("NexusTag") then
            target.NexusTag:Destroy()
        end
    end

    entry = {
        hl = hl,
        isPriorityHighlight = isPriorityHighlight,
        gui = gui,
        lbl = lbl,
        lastText = nil,
        lastHlEnabled = nil,
        lastGuiEnabled = nil
    }
    VisualRegistry[target] = entry
    return entry
end

local function setVisualState(entry, target, anchorPart, wantHl, wantGui, col, txt, isClose)
    if not entry or not entry.hl or not entry.hl.Parent then return end

    if wantHl then
        if entry.isPriorityHighlight then
            suppressCompetingHighlights(target, entry.hl)
            if isClose then
                local oldHl = entry.hl
                local freshHl = Instance.new("Highlight")
                freshHl.Name = "NexusHL"
                freshHl.Adornee = target
                freshHl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                freshHl.FillColor = col
                freshHl.OutlineColor = col
                freshHl.FillTransparency = 0.34
                freshHl.OutlineTransparency = 0.0
                freshHl.Enabled = true
                freshHl.Parent = PriorityHighlightFolder
                entry.hl = freshHl
                if oldHl then
                    pcall(function() oldHl:Destroy() end)
                end
                entry.lastHlEnabled = true
                return
            end

            if entry.hl.Parent ~= PriorityHighlightFolder then
                entry.hl.Parent = PriorityHighlightFolder
            end
        end

        if entry.hl.Adornee ~= target then entry.hl.Adornee = target end
        entry.hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
        entry.hl.FillTransparency = entry.isPriorityHighlight and 0.36 or 0.50
        entry.hl.OutlineTransparency = entry.isPriorityHighlight and 0.0 or 0.05
        if entry.lastHlEnabled ~= true then
            entry.hl.Enabled = true
            entry.lastHlEnabled = true
        end
        entry.hl.FillColor = col
        entry.hl.OutlineColor = col
    else
        if entry.lastHlEnabled ~= false then
            entry.hl.Enabled = false
            entry.lastHlEnabled = false
        end
    end

    if entry.gui and entry.lbl then
        if wantGui then
            local tagParent = (anchorPart and anchorPart.Parent) and anchorPart or target
            if entry.gui.Parent ~= tagParent then entry.gui.Parent = tagParent end
            if entry.gui.Adornee ~= tagParent then entry.gui.Adornee = tagParent end
            if entry.lastGuiEnabled ~= true then
                entry.gui.Enabled = true
                entry.lastGuiEnabled = true
            end
            entry.lbl.TextColor3 = col
            if entry.lastText ~= txt then
                entry.lbl.Text = txt
                entry.lastText = txt
            end
        else
            if entry.lastGuiEnabled ~= false then
                entry.gui.Enabled = false
                entry.lastGuiEnabled = false
            end
        end
    end
end

local function hideVisual(target)
    local entry = VisualRegistry[target]
    if entry then
        setVisualState(entry, target, nil, false, false, nil, nil, false)
    end
end

local function destroyVisual(target)
    local entry = VisualRegistry[target]
    if entry then
        if entry.hl then pcall(function() entry.hl:Destroy() end) end
        if entry.gui then pcall(function() entry.gui:Destroy() end) end
        VisualRegistry[target] = nil
    end
    if target then
        destroyPlayerVisual(target)
        local tag = target:FindFirstChild("NexusTag")
        if tag then pcall(function() tag:Destroy() end) end
    end
end

local function rebuildCaches()
    local mystery, regs, safes, atms, dealers, rebels, crates, ground, trash = {}, {}, {}, {}, {}, {}, {}, {}, {}
    local map = Workspace:FindFirstChild("Map")
    local filter = Workspace:FindFirstChild("Filter")

    local mysteryFolder = map and map:FindFirstChild("MysteryBoxes")
    if mysteryFolder then
        for _, child in ipairs(mysteryFolder:GetChildren()) do
            if child:IsA("Model") or child:IsA("BasePart") then
                table.insert(mystery, child)
            end
        end
    end

    local bred = map and map:FindFirstChild("BredMakurz")
    if bred then
        for _, child in ipairs(bred:GetChildren()) do
            if child:IsA("Model") then
                local low = child.Name:lower()
                if low:find("register", 1, true) then
                    table.insert(regs, child)
                elseif low:find("safe", 1, true) or low:find("vault", 1, true) then
                    table.insert(safes, child)
                end
            end
        end
    end

    local atmFolder = map and map:FindFirstChild("ATMz")
    if atmFolder then
        for _, child in ipairs(atmFolder:GetChildren()) do
            if child:IsA("Model") or child:IsA("BasePart") then
                table.insert(atms, child)
            end
        end
    end

    local shopz = map and map:FindFirstChild("Shopz")
    if shopz then
        for _, child in ipairs(shopz:GetChildren()) do
            if child:IsA("Model") and child.Name:lower():find("dealer", 1, true) then
                table.insert(dealers, child)
            end
        end
    end

    if filter then
        local piles = filter:FindFirstChild("SpawnedPiles")
        if piles then
            for _, c in ipairs(piles:GetChildren()) do
                if c:IsA("Model") or c:IsA("BasePart") then
                    if c.Name:match("^C%d+") then
                        table.insert(crates, c)
                    elseif c.Name:match("^S%d+") then
                        table.insert(trash, c)
                    else
                        table.insert(ground, c)
                    end
                end
            end
        end

        local sTools = filter:FindFirstChild("SpawnedTools")
        if sTools then
            for _, c in ipairs(sTools:GetChildren()) do
                if c:IsA("Model") or c:IsA("BasePart") or c:IsA("Tool") then
                    table.insert(ground, c)
                end
            end
        end

        local sBread = filter:FindFirstChild("SpawnedBread")
        if sBread then
            for _, c in ipairs(sBread:GetChildren()) do
                if c:IsA("Model") or c:IsA("BasePart") or c:IsA("Tool") then
                    table.insert(ground, c)
                end
            end
        end
    end

    Cache.MysteryBoxes, Cache.Registers, Cache.Safes, Cache.ATMs, Cache.Dealers = mystery, regs, safes, atms, dealers
    Cache.RebelDealers, Cache.Crates, Cache.GroundItems, Cache.Trash = rebels, crates, ground, trash
end

local running = true

rebuildCaches()

task.spawn(function()
    while running do
        rebuildCaches()
        task.wait(2.5)
    end
end)

local candidateBuffer = {}
local partsBuffer = {}

local function processCategory(list, cfg, kind, colFn, maxPerCategory, noTag, prioritizeClose)
    if not cfg.Enabled then
        for i = 1, #list do
            hideVisual(list[i])
        end
        return
    end

    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local myPos = root and root.Position
    if not myPos then
        for i = 1, #list do
            hideVisual(list[i])
        end
        return
    end

    table.clear(candidateBuffer)
    local maxDist = cfg.Distance
    local maxDistSq = maxDist * maxDist
    local mx, my, mz = myPos.X, myPos.Y, myPos.Z

    for i = 1, #list do
        local o = list[i]
        if o.Parent then
            local meta = getMeta(o, kind)
            local part = meta.anchor
            if not part or not part.Parent then
                part = resolveAnchorPart(o)
                meta.anchor = part
            end
            if part and part.Parent then
                local pPos = part.Position
                local dx, dy, dz = pPos.X - mx, pPos.Y - my, pPos.Z - mz
                local dSq = dx * dx + dy * dy + dz * dz
                if dSq <= maxDistSq then
                    local dist = math.floor(math.sqrt(dSq))
                    table.insert(candidateBuffer, { obj = o, meta = meta, dist = dist })
                else
                    hideVisual(o)
                end
            else
                hideVisual(o)
            end
        else
            hideVisual(o)
        end
    end

    if prioritizeClose and #candidateBuffer > 1 then
        table.sort(candidateBuffer, function(a, b)
            return a.dist < b.dist
        end)
    end

    local shown = 0
    local limit = maxPerCategory or 35
    local isPriorityCategory = (kind == "crate" or kind == "ground")

    for i = 1, #candidateBuffer do
        local item = candidateBuffer[i]
        if shown >= limit and not (isPriorityCategory and item.dist <= 15) then
            hideVisual(item.obj)
        else
            shown += 1
            local entry = ensureVisual(item.obj, item.meta.anchor, noTag, isPriorityCategory)
            if entry then
                local col = colFn and colFn(item.obj, item.meta) or cfg.Color
                local allowHl = isPriorityCategory or (activeHighlightBudget < MAX_TOTAL_HIGHLIGHTS)
                if not isPriorityCategory and allowHl then
                    activeHighlightBudget += 1
                end

                local showLabel = false
                local txt = ""
                if not noTag then
                    showLabel = cfg.ShowDist or (item.meta.label ~= "")
                    if showLabel then
                        txt = cfg.ShowDist and (item.meta.label .. " [" .. item.dist .. "]") or item.meta.label
                    end
                end

                local isClose = isPriorityCategory and (item.dist <= 6)
                setVisualState(entry, item.obj, item.meta.anchor, allowHl, showLabel, col, txt, isClose)
            end
        end
    end
end

local inputConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    local kc = input.KeyCode
    if kc == Enum.KeyCode.Unknown then return end

    if listeningForKey then
        listeningForKey = false
        activeToggleKey = kc
        Settings.ToggleKeyName = kc.Name
        local label = formatKeyLabel(kc)
        BindBtn.Text = "TECLA: " .. label .. " (CLIQUE P/ TROCAR)"
        BindBtn.TextColor3 = Color3.fromRGB(233, 213, 255)
        KeyStatus.Text = "Tecla atual: " .. label
        KeyStatus.TextColor3 = Color3.fromRGB(192, 132, 252)
        return
    end

    if gameProcessed and kc ~= Enum.KeyCode.RightAlt then return end
    if activeToggleKey and kc == activeToggleKey then
        setUIVisible(not MainFrame.Visible)
    end
end)

local accum = 0
local renderConn = RunService.RenderStepped:Connect(function(dt)
    if Settings.NightVision then
        applyNightVisionFrame()
    end

    accum += dt
    if accum < 0.032 then return end
    accum = 0

    activeHighlightBudget = 0
    Settings.Trash.Color = computeScrapColor()

    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    local myPos = root and root.Position

    if Settings.Players.Enabled then
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local pChar = plr.Character
                if not isPlayerAllowedByFilter(plr) then
                    destroyPlayerVisual(pChar)
                else
                    local pRoot = pChar:FindFirstChild("HumanoidRootPart") or pChar:FindFirstChild("Head")
                    local pHum = pChar:FindFirstChildOfClass("Humanoid")
                    if myPos and pRoot and pHum and pHum.Health > 0 then
                        local dist = math.floor((pRoot.Position - myPos).Magnitude)
                        if dist <= Settings.Players.Distance then
                            local hl = pChar:FindFirstChild("NexusHL")
                            if not hl then
                                hl = Instance.new("Highlight")
                                hl.Name = "NexusHL"
                            end
                            hl.Adornee = pChar
                            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                            hl.FillTransparency = 0.55
                            hl.OutlineTransparency = 0.05
                            hl.FillColor = Settings.Players.Color
                            hl.OutlineColor = Settings.Players.Color
                            hl.Enabled = true
                            hl.Parent = pChar

                            local gui = pChar:FindFirstChild("NexusPlrGui")
                            if not gui then
                                gui = Instance.new("BillboardGui")
                                gui.Name = "NexusPlrGui"
                            end
                            gui.Size = UDim2.new(0, 220, 0, 32)
                            gui.Adornee = pRoot
                            gui.StudsOffset = Vector3.new(0, 3.2, 0)
                            gui.AlwaysOnTop = true
                            gui.MaxDistance = math.huge
                            gui.ClipsDescendants = false
                            gui.Enabled = true
                            gui.Parent = pChar

                            local lbl = gui:FindFirstChild("Lbl")
                            if not lbl then
                                lbl = Instance.new("TextLabel")
                                lbl.Name = "Lbl"
                            end
                            lbl.Size = UDim2.new(1, 0, 1, 0)
                            lbl.BackgroundTransparency = 1
                            lbl.TextColor3 = hl.OutlineColor
                            lbl.TextSize = 12
                            lbl.Font = Enum.Font.GothamBold
                            lbl.TextStrokeTransparency = 0.25
                            lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                            lbl.ClipsDescendants = false
                            lbl.Parent = gui

                            table.clear(partsBuffer)
                            if Settings.Players.ShowName then table.insert(partsBuffer, plr.Name) end
                            if Settings.Players.ShowHealth then table.insert(partsBuffer, math.floor(pHum.Health) .. "HP") end
                            if Settings.Players.ShowWeapon then
                                local tool = pChar:FindFirstChildOfClass("Tool")
                                table.insert(partsBuffer, tool and tool.Name or "Sem Arma")
                            end
                            if Settings.Players.ShowDist then
                                table.insert(partsBuffer, dist .. "s")
                            end
                            local nextTxt = table.concat(partsBuffer, " | ")
                            if lbl.Text ~= nextTxt then
                                lbl.Text = nextTxt
                            end
                        else
                            destroyPlayerVisual(pChar)
                        end
                    else
                        destroyPlayerVisual(pChar)
                    end
                end
            end
        end
    end

    processCategory(Cache.MysteryBoxes, Settings.MysteryBoxes, "mystery", function() return Settings.MysteryBoxes.Color end, 1, false, true)
    processCategory(Cache.GroundItems, Settings.GroundItems, "ground", function(_, m) return m.isCash and Settings.GroundItems.CashColor or Settings.GroundItems.Color end, 35, false, true)
    processCategory(Cache.Registers, Settings.Registers, "register", function(_, m) return getStateColor(Settings.Registers, m) end, 25, false, false)
    processCategory(Cache.Safes, Settings.Safes, "safe", function(_, m) return getStateColor(Settings.Safes, m) end, 25, false, false)
    processCategory(Cache.ATMs, Settings.ATMs, "atm", nil, 25, false, false)
    processCategory(Cache.Dealers, Settings.Dealers, "dealer", nil, 20, false, false)
    processCategory(Cache.RebelDealers, Settings.RebelDealers, "dealer", nil, 20, false, false)
    processCategory(Cache.Crates, Settings.Crates, "crate", function() return Settings.Crates.Color end, 25, true, true)
    processCategory(Cache.Trash, Settings.Trash, "scrap", function() return Settings.Trash.Color end, 35, false, true)
end)

ExitBtn.MouseButton1Click:Connect(function()
    running = false
    setNightVision(false)
    if nvgCorrection then pcall(function() nvgCorrection:Destroy() end) end
    if renderConn then renderConn:Disconnect() end
    if inputConn then inputConn:Disconnect() end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr.Character then destroyPlayerVisual(plr.Character) end
    end
    for target in pairs(VisualRegistry) do
        destroyVisual(target)
    end
    ScreenGui:Destroy()
end)
