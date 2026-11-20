--[[
    NEXUS UI | CRIMINALITY (VERSÃO EXECUTOR COMPLETA)
    - Ícone: Coelho estilizado em alto contraste (P&B) via UI nativa
    - Sem botão "OCULTAR UI": botão "—" na barra de título faz a ocultação e "FECHAR E LIMPAR" ocupa a maior parte da faixa
    - Crates & Ground Items: Highlight na malha real + supressão de Highlight nativo perto
    - Mystery Boxes: Workspace.Map.MysteryBoxes (ex.: HALLOWS_MysteryBox - mais próxima)
    - Registers / Safes / ATMs / Dealers / Trash / Night Vision
    - Rebel Dealer ESP: Desabilitado e marcado como EM MANUTENÇÃO
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
    RebelDealers= { Enabled = false, Color = Color3.fromRGB(255, 0, 0), Distance = 250, ShowDist = true, Maintenance = true },
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
ScreenGui.Parent
