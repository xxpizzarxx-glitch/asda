-- ╔══════════════════════════════════════════╗
-- ║     🍕 PIZZA HUB v3.5 (COM ESP)        ║
-- ║  Wallhack · Aimbot · Triggerbot · ESP   ║
-- ║  F1 = abrir/fechar  SHIFT = mirar       ║
-- ╚══════════════════════════════════════════╝

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local HttpService      = game:GetService("HttpService") -- para JSON no ESP (opcional)

local LP     = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ══════════════════════════════════════════
--  CONFIGURAÇÕES
-- ══════════════════════════════════════════
local CFG = {
    WH_On         = true,
    WH_EnemyColor = Color3.fromRGB(255, 60, 60),
    WH_AllyColor  = Color3.fromRGB(60, 220, 100),

    AB_On         = true,
    AB_Smooth     = 0.3,   -- 0 = instantâneo, 1 = muito lento
    AB_FOV        = 150,   -- pixels do centro da tela
    AB_Part       = "Head",
    AB_TeamCheck  = true,
    AB_HealthCheck= true,

    TB_On         = false,
    TB_Px         = 20,    -- pixels do centro pra atirar
    TB_Delay      = 0.12,

    ESP_On        = true,
    ESP_ShowName  = true,
    ESP_ShowHP    = true,
    ESP_ShowDist  = true,
}

-- ══════════════════════════════════════════
--  WALLHACK  (Highlight nativo)
-- ══════════════════════════════════════════
local highlights = {}   -- [player] = Highlight instance

local function AddHighlight(p)
    if p == LP then return end
    -- remove antigo se existir
    if highlights[p] then
        highlights[p]:Destroy()
        highlights[p] = nil
    end
    local char = p.Character
    if not char then return end

    local hl = Instance.new("Highlight")
    hl.Adornee            = char
    hl.DepthMode          = Enum.HighlightDepthMode.AlwaysOnTop
    hl.FillTransparency   = 0.5
    hl.OutlineTransparency= 0
    hl.Enabled            = CFG.WH_On

    local isEnemy = CFG.AB_TeamCheck and (p.Team ~= LP.Team)
    hl.FillColor    = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor
    hl.OutlineColor = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor

    hl.Parent      = char
    highlights[p]  = hl
end

local function RemoveHighlight(p)
    if highlights[p] then
        highlights[p]:Destroy()
        highlights[p] = nil
    end
end

local function RefreshAllHighlights()
    for p, hl in pairs(highlights) do
        hl.Enabled = CFG.WH_On
        local isEnemy = CFG.AB_TeamCheck and (p.Team ~= LP.Team)
        hl.FillColor    = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor
        hl.OutlineColor = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor
    end
end

-- liga highlight em todos que já estão no jogo
for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LP then
        AddHighlight(p)
        p.CharacterAdded:Connect(function()
            task.wait(0.5)
            AddHighlight(p)
        end)
    end
end

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(0.5)
        AddHighlight(p)
    end)
end)

Players.PlayerRemoving:Connect(function(p)
    RemoveHighlight(p)
end)

-- ══════════════════════════════════════════
--  ESP  (nome, HP, distância)
-- ══════════════════════════════════════════
local espBinds = {}   -- [player] = {billboard, nameLabel, hpLabel, distLabel}

local function CreateESP(p)
    if p == LP then return end
    local char = p.Character
    if not char then return end
    local head = char:FindFirstChild("Head")
    if not head then return end

    local bill = Instance.new("BillboardGui")
    bill.Name           = "ESP"
    bill.Size           = UDim2.new(0, 200, 0, 60)
    bill.StudsOffset    = Vector3.new(0, 2.5, 0)
    bill.AlwaysOnTop    = true
    bill.MaxDistance    = 500
    bill.Adornee        = head
    bill.Parent         = char

    local nameLbl = Instance.new("TextLabel")
    nameLbl.Size                = UDim2.new(1, 0, 0, 18)
    nameLbl.BackgroundTransparency = 1
    nameLbl.TextColor3          = Color3.fromRGB(255, 255, 255)
    nameLbl.Font                = Enum.Font.GothamBold
    nameLbl.TextSize            = 13
    nameLbl.TextStrokeTransparency = 0.6
    nameLbl.Text                = CFG.ESP_ShowName and p.DisplayName or ""
    nameLbl.Parent              = bill

    local hpLbl = Instance.new("TextLabel")
    hpLbl.Size                  = UDim2.new(1, 0, 0, 18)
    hpLbl.Position              = UDim2.new(0, 0, 0, 20)
    hpLbl.BackgroundTransparency = 1
    hpLbl.TextColor3            = Color3.fromRGB(255, 100, 100)
    hpLbl.Font                  = Enum.Font.Gotham
    hpLbl.TextSize              = 12
    hpLbl.TextStrokeTransparency = 0.6
    hpLbl.Text                  = ""
    hpLbl.Parent                = bill

    local distLbl = Instance.new("TextLabel")
    distLbl.Size                = UDim2.new(1, 0, 0, 18)
    distLbl.Position            = UDim2.new(0, 0, 0, 40)
    distLbl.BackgroundTransparency = 1
    distLbl.TextColor3          = Color3.fromRGB(200, 200, 200)
    distLbl.Font                = Enum.Font.Gotham
    distLbl.TextSize            = 11
    distLbl.TextStrokeTransparency = 0.6
    distLbl.Text                = ""
    distLbl.Parent              = bill

    espBinds[p] = {billboard = bill, nameLabel = nameLbl, hpLabel = hpLbl, distLabel = distLbl}
end

local function RemoveESP(p)
    if espBinds[p] then
        espBinds[p].billboard:Destroy()
        espBinds[p] = nil
    end
end

local function RefreshESP()
    if not CFG.ESP_On then
        for p, data in pairs(espBinds) do
            data.billboard.Enabled = false
        end
        return
    end
    local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    for p, data in pairs(espBinds) do
        local char = p.Character
        if not char then
            data.billboard.Enabled = false
            continue
        end
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        data.billboard.Enabled = true

        -- nome
        data.nameLabel.Text = CFG.ESP_ShowName and p.DisplayName or ""

        -- HP
        if CFG.ESP_ShowHP and hum then
            local hp = math.floor(hum.Health)
            data.hpLabel.Text = "HP: " .. hp
            if hp > 60 then
                data.hpLabel.TextColor3 = Color3.fromRGB(60, 220, 100)
            elseif hp > 30 then
                data.hpLabel.TextColor3 = Color3.fromRGB(255, 200, 50)
            else
                data.hpLabel.TextColor3 = Color3.fromRGB(255, 60, 60)
            end
        else
            data.hpLabel.Text = ""
        end

        -- distância
        if CFG.ESP_ShowDist and myRoot and hrp then
            local dist = math.floor((myRoot.Position - hrp.Position).Magnitude)
            data.distLabel.Text = dist .. " st"
        else
            data.distLabel.Text = ""
        end
    end
end

-- conecta ESP em todos os players
for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LP then
        CreateESP(p)
        p.CharacterAdded:Connect(function()
            task.wait(0.5)
            RemoveESP(p) -- limpa antigo
            CreateESP(p)
        end)
    end
end

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(0.5)
        RemoveESP(p)
        CreateESP(p)
    end)
end)

Players.PlayerRemoving:Connect(function(p)
    RemoveHighlight(p)
    RemoveESP(p)
end)

-- ══════════════════════════════════════════
--  AIMBOT
-- ══════════════════════════════════════════
local aiming    = false
local tbTimer   = 0

local function GetTarget()
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local best, bestDist = nil, CFG.AB_FOV

    for _, p in ipairs(Players:GetPlayers()) do
        if p == LP then continue end
        if CFG.AB_TeamCheck and p.Team == LP.Team then continue end
        local char = p.Character
        if not char then continue end
        local hum  = char:FindFirstChildOfClass("Humanoid")
        if CFG.AB_HealthCheck and (not hum or hum.Health <= 0) then continue end
        local part = char:FindFirstChild(CFG.AB_Part) or char:FindFirstChild("HumanoidRootPart")
        if not part then continue end

        local sp, onScreen = Camera:WorldToViewportPoint(part.Position)
        if not onScreen or sp.Z <= 0 then continue end

        local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
        if d < bestDist then
            bestDist = d
            best     = part
        end
    end
    return best
end

local function AimbotStep()
    if not CFG.AB_On or not aiming then return end
    local part = GetTarget()
    if not part then return end

    local pos  = part.Position
    local cf   = Camera.CFrame
    local goal = CFrame.lookAt(cf.Position, pos)
    Camera.CFrame = cf:Lerp(goal, 1 - CFG.AB_Smooth)
end

-- ══════════════════════════════════════════
--  TRIGGERBOT
-- ══════════════════════════════════════════
local function TriggerbotStep(dt)
    if not CFG.TB_On then return end
    tbTimer = tbTimer - dt
    if tbTimer > 0 then return end

    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    for _, p in ipairs(Players:GetPlayers()) do
        if p == LP then continue end
        if CFG.TB_TeamCheck and p.Team == LP.Team then continue end
        local char = p.Character
        if not char then continue end
        local hum  = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then continue end
        local head = char:FindFirstChild("Head")
        if not head then continue end

        local sp, on = Camera:WorldToViewportPoint(head.Position)
        if not on or sp.Z <= 0 then continue end
        if (Vector2.new(sp.X, sp.Y) - center).Magnitude <= CFG.TB_Px then
            -- simula clique
            local v = game:GetService("VirtualUser")
            v:Button1Down(center, Camera.CFrame)
            task.delay(0.05, function() v:Button1Up(center, Camera.CFrame) end)
            tbTimer = CFG.TB_Delay
            break
        end
    end
end

-- ══════════════════════════════════════════
--  CIRCLE FOV (Drawing)
-- ══════════════════════════════════════════
local fovCircle      = Drawing.new("Circle")
fovCircle.Color      = Color3.fromRGB(255, 140, 0)
fovCircle.Thickness  = 1.5
fovCircle.NumSides   = 60
fovCircle.Filled     = false
fovCircle.ZIndex     = 5
fovCircle.Visible    = false

-- ══════════════════════════════════════════
--  GUI — PIZZA HUB
-- ══════════════════════════════════════════
-- Remove gui antiga se existir
pcall(function()
    game:GetService("CoreGui"):FindFirstChild("PizzaHub"):Destroy()
end)

local SG = Instance.new("ScreenGui")
SG.Name           = "PizzaHub"
SG.ResetOnSpawn   = false
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SG.IgnoreGuiInset = true
SG.Parent         = game:GetService("CoreGui")

-- Cores
local A1  = Color3.fromRGB(255, 140, 0)   -- laranja
local A2  = Color3.fromRGB(220, 40,  40)  -- vermelho
local BG  = Color3.fromRGB(10,  10,  15)
local CAR = Color3.fromRGB(20,  20,  30)
local TXT = Color3.fromRGB(230, 230, 240)
local SUB = Color3.fromRGB(130, 130, 155)
local TOG = Color3.fromRGB(35,  35,  50)
local GRN = Color3.fromRGB(60,  220, 100)
local TI  = TweenInfo.new(0.18, Enum.EasingStyle.Quad)

-- Atalho rápido para criar instâncias
local function N(c, props, parent)
    local i = Instance.new(c)
    for k,v in pairs(props) do i[k]=v end
    if parent then i.Parent = parent end
    return i
end

-- ── Frame principal ──
local Main = N("Frame", {
    Size             = UDim2.new(0, 340, 0, 440), -- aumentei um pouco
    Position         = UDim2.new(0, 60, 0, 60),
    BackgroundColor3 = BG,
    BorderSizePixel  = 0,
    Active           = true,
    Draggable        = true,
    ClipsDescendants = true,
}, SG)
N("UICorner", {CornerRadius = UDim.new(0, 12)}, Main)
N("UIStroke", {Color = A1, Thickness = 1.2, Transparency = 0.3}, Main)

-- Barra topo colorida
local topBar = N("Frame", {
    Size             = UDim2.new(1, 0, 0, 4),
    BackgroundColor3 = A1,
    BorderSizePixel  = 0,
}, Main)
N("UICorner", {CornerRadius = UDim.new(0, 12)}, topBar)
N("UIGradient", {Color = ColorSequence.new(A1, A2), Rotation = 0}, topBar)

-- Título
N("TextLabel", {
    Size             = UDim2.new(1, -16, 0, 44),
    Position         = UDim2.new(0, 14, 0, 6),
    BackgroundTransparency = 1,
    Text             = "🍕  PIZZA HUB",
    TextColor3       = TXT,
    Font             = Enum.Font.GothamBold,
    TextSize         = 17,
    TextXAlignment   = Enum.TextXAlignment.Left,
}, Main)

N("TextLabel", {
    Size             = UDim2.new(1, -16, 0, 16),
    Position         = UDim2.new(0, 14, 0, 42),
    BackgroundTransparency = 1,
    Text             = "F1 fechar  ·  SHIFT mirar",
    TextColor3       = SUB,
    Font             = Enum.Font.Gotham,
    TextSize         = 11,
    TextXAlignment   = Enum.TextXAlignment.Left,
}, Main)

-- Linha divisória
N("Frame", {
    Size             = UDim2.new(1, -28, 0, 1),
    Position         = UDim2.new(0, 14, 0, 62),
    BackgroundColor3 = Color3.fromRGB(40, 40, 60),
    BorderSizePixel  = 0,
}, Main)

-- Scroll com os controles
local Scroll = N("ScrollingFrame", {
    Size                 = UDim2.new(1, 0, 1, -68),
    Position             = UDim2.new(0, 0, 0, 68),
    BackgroundTransparency = 1,
    ScrollBarThickness   = 3,
    ScrollBarImageColor3 = A1,
    BorderSizePixel      = 0,
    CanvasSize           = UDim2.new(0, 0, 0, 0),
    AutomaticCanvasSize  = Enum.AutomaticSize.Y,
    ClipsDescendants     = true,
}, Main)
N("UIPadding",    {PaddingLeft=UDim.new(0,12), PaddingRight=UDim.new(0,12), PaddingTop=UDim.new(0,8), PaddingBottom=UDim.new(0,10)}, Scroll)
N("UIListLayout", {Padding=UDim.new(0,8), SortOrder=Enum.SortOrder.LayoutOrder}, Scroll)

-- ── helpers de componentes ──
local function SecLabel(txt, order)
    local f = N("Frame", {
        Size             = UDim2.new(1, 0, 0, 26),
        BackgroundTransparency = 1,
        LayoutOrder      = order,
    }, Scroll)
    N("TextLabel", {
        Size             = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text             = txt,
        TextColor3       = A1,
        Font             = Enum.Font.GothamBold,
        TextSize         = 11,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, f)
    N("Frame", {
        Size             = UDim2.new(1, 0, 0, 1),
        Position         = UDim2.new(0, 0, 1, 0),
        BackgroundColor3 = A1,
        BackgroundTransparency = 0.65,
        BorderSizePixel  = 0,
    }, f)
end

local function MakeToggle(label, initVal, onChange, order)
    local row = N("Frame", {
        Size             = UDim2.new(1, 0, 0, 44),
        BackgroundColor3 = CAR,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, Scroll)
    N("UICorner", {CornerRadius = UDim.new(0, 8)}, row)

    N("TextLabel", {
        Size             = UDim2.new(1, -60, 1, 0),
        Position         = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text             = label,
        TextColor3       = TXT,
        Font             = Enum.Font.Gotham,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, row)

    local pill = N("Frame", {
        Size             = UDim2.new(0, 42, 0, 22),
        Position         = UDim2.new(1, -50, 0.5, -11),
        BackgroundColor3 = initVal and A1 or TOG,
        BorderSizePixel  = 0,
    }, row)
    N("UICorner", {CornerRadius = UDim.new(1, 0)}, pill)

    local knob = N("Frame", {
        Size             = UDim2.new(0, 17, 0, 17),
        Position         = initVal and UDim2.new(1,-20,0.5,-8) or UDim2.new(0,3,0.5,-8),
        BackgroundColor3 = Color3.new(1,1,1),
        BorderSizePixel  = 0,
    }, pill)
    N("UICorner", {CornerRadius = UDim.new(1, 0)}, knob)

    local val = initVal
    local btn = N("TextButton", {
        Size             = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text             = "",
        ZIndex           = 5,
    }, row)

    local function SetState(v)
        val = v
        TweenService:Create(pill, TI, {BackgroundColor3 = v and A1 or TOG}):Play()
        TweenService:Create(knob, TI, {Position = v and UDim2.new(1,-20,0.5,-8) or UDim2.new(0,3,0.5,-8)}):Play()
    end

    btn.MouseButton1Click:Connect(function()
        SetState(not val)
        onChange(val)
    end)

    return SetState
end

local function MakeSlider(label, min, max, cur, decimals, onChange, order)
    local card = N("Frame", {
        Size             = UDim2.new(1, 0, 0, 60),
        BackgroundColor3 = CAR,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, Scroll)
    N("UICorner", {CornerRadius = UDim.new(0, 8)}, card)

    N("TextLabel", {
        Size             = UDim2.new(1, -70, 0, 22),
        Position         = UDim2.new(0, 12, 0, 6),
        BackgroundTransparency = 1,
        Text             = label,
        TextColor3       = TXT,
        Font             = Enum.Font.Gotham,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, card)

    local fmt   = "%." .. decimals .. "f"
    local vLbl  = N("TextLabel", {
        Size             = UDim2.new(0, 58, 0, 22),
        Position         = UDim2.new(1, -66, 0, 6),
        BackgroundTransparency = 1,
        Text             = string.format(fmt, cur),
        TextColor3       = A1,
        Font             = Enum.Font.GothamBold,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Right,
    }, card)

    local track = N("Frame", {
        Size             = UDim2.new(1, -24, 0, 5),
        Position         = UDim2.new(0, 12, 0, 40),
        BackgroundColor3 = TOG,
        BorderSizePixel  = 0,
    }, card)
    N("UICorner", {CornerRadius = UDim.new(1, 0)}, track)

    local t0   = (cur - min) / (max - min)
    local fill = N("Frame", {
        Size             = UDim2.new(t0, 0, 1, 0),
        BackgroundColor3 = A1,
        BorderSizePixel  = 0,
    }, track)
    N("UICorner", {CornerRadius = UDim.new(1, 0)}, fill)

    -- Thumb
    local thumb = N("Frame", {
        Size             = UDim2.new(0, 14, 0, 14),
        Position         = UDim2.new(t0, -7, 0.5, -7),
        BackgroundColor3 = Color3.new(1,1,1),
        BorderSizePixel  = 0,
        ZIndex           = 3,
    }, track)
    N("UICorner", {CornerRadius = UDim.new(1, 0)}, thumb)

    local dragging = false
    local function UpdateSlider(x)
        local ax = track.AbsolutePosition.X
        local aw = track.AbsoluteSize.X
        local t  = math.clamp((x - ax) / aw, 0, 1)
        local v  = min + t * (max - min)
        if decimals == 0 then v = math.floor(v + 0.5) end
        fill.Size      = UDim2.new(t, 0, 1, 0)
        thumb.Position = UDim2.new(t, -7, 0.5, -7)
        vLbl.Text      = string.format(fmt, v)
        onChange(v)
    end

    track.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            UpdateSlider(i.Position.X)
        end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then
            UpdateSlider(i.Position.X)
        end
    end)
end

-- ── Info card do alvo ──
local function MakeInfoCard(order)
    local card = N("Frame", {
        Size             = UDim2.new(1, 0, 0, 52),
        BackgroundColor3 = CAR,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, Scroll)
    N("UICorner", {CornerRadius = UDim.new(0, 8)}, card)
    N("UIStroke", {Color = A1, Thickness = 0.8, Transparency = 0.4}, card)

    N("TextLabel", {
        Size             = UDim2.new(0, 80, 0, 18),
        Position         = UDim2.new(0, 12, 0, 7),
        BackgroundTransparency = 1,
        Text             = "🎯 ALVO ATUAL",
        TextColor3       = A1,
        Font             = Enum.Font.GothamBold,
        TextSize         = 11,
    }, card)

    local nLbl = N("TextLabel", {
        Size             = UDim2.new(1, -20, 0, 18),
        Position         = UDim2.new(0, 12, 0, 28),
        BackgroundTransparency = 1,
        Text             = "nenhum",
        TextColor3       = SUB,
        Font             = Enum.Font.Gotham,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, card)

    local dLbl = N("TextLabel", {
        Size             = UDim2.new(0, 80, 0, 18),
        Position         = UDim2.new(1, -90, 0, 7),
        BackgroundTransparency = 1,
        Text             = "",
        TextColor3       = GRN,
        Font             = Enum.Font.GothamBold,
        TextSize         = 12,
        TextXAlignment   = Enum.TextXAlignment.Right,
    }, card)

    return nLbl, dLbl
end

-- ══════════════════════════════════════════
--  MONTA OS CONTROLES
-- ══════════════════════════════════════════

SecLabel("◈  WALLHACK", 1)

MakeToggle("👁  Highlight (ver pela parede)", CFG.WH_On, function(v)
    CFG.WH_On = v
    RefreshAllHighlights()
end, 2)

SecLabel("◈  ESP", 3)

MakeToggle("📋  ESP (nome, HP, distância)", CFG.ESP_On, function(v)
    CFG.ESP_On = v
    RefreshESP()
end, 4)

SecLabel("◈  AIMBOT  (segurar SHIFT)", 5)

MakeToggle("🤖  Aimbot ligado", CFG.AB_On, function(v)
    CFG.AB_On = v
    fovCircle.Visible = v
end, 6)

MakeSlider("Suavidade (0=rápido, 1=lento)", 0, 0.99, CFG.AB_Smooth, 2, function(v)
    CFG.AB_Smooth = v
end, 7)

MakeSlider("FOV (pixels do centro)", 20, 500, CFG.AB_FOV, 0, function(v)
    CFG.AB_FOV = v
end, 8)

SecLabel("◈  TRIGGERBOT", 9)

MakeToggle("🔫  Triggerbot ligado", CFG.TB_On, function(v)
    CFG.TB_On = v
end, 10)

MakeSlider("Threshold (pixels)", 5, 100, CFG.TB_Px, 0, function(v)
    CFG.TB_Px = v
end, 11)

MakeSlider("Delay entre tiros (s)", 0.02, 0.5, CFG.TB_Delay, 2, function(v)
    CFG.TB_Delay = v
end, 12)

SecLabel("◈  CONFIGURAÇÕES", 13)

MakeToggle("⚔  Team Check (não mirar aliados)", CFG.AB_TeamCheck, function(v)
    CFG.AB_TeamCheck = v
    CFG.TB_TeamCheck = v
    RefreshAllHighlights()
end, 14)

MakeToggle("❤  Health Check (não mirar mortos)", CFG.AB_HealthCheck, function(v)
    CFG.AB_HealthCheck = v
end, 15)

SecLabel("◈  STATUS", 16)
local targetName, targetDist = MakeInfoCard(17)

-- ══════════════════════════════════════════
--  INPUTS
-- ══════════════════════════════════════════
UserInputService.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.F1 then
        Main.Visible = not Main.Visible
    elseif i.KeyCode == Enum.KeyCode.LeftShift or i.KeyCode == Enum.KeyCode.RightShift then
        aiming = true
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.KeyCode == Enum.KeyCode.LeftShift or i.KeyCode == Enum.KeyCode.RightShift then
        aiming = false
    end
end)

-- ══════════════════════════════════════════
--  LOOP PRINCIPAL
-- ══════════════════════════════════════════
local tick = 0

RunService.RenderStepped:Connect(function(dt)
    tick = tick + 1

    -- Aimbot
    AimbotStep()

    -- Triggerbot
    TriggerbotStep(dt)

    -- FOV circle
    if CFG.AB_On then
        local vp = Camera.ViewportSize
        fovCircle.Visible  = true
        fovCircle.Position = Vector2.new(vp.X * 0.5, vp.Y * 0.5)
        fovCircle.Radius   = CFG.AB_FOV
    else
        fovCircle.Visible = false
    end

    -- Atualiza ESP
    if tick % 10 == 0 then
        RefreshESP()
    end

    -- Atualiza info do alvo (a cada 10 frames)
    if tick % 10 == 0 then
        local part = GetTarget()
        if part then
            local p = Players:GetPlayerFromCharacter(part.Parent)
            if p then
                local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
                local dist   = myRoot and math.floor((myRoot.Position - part.Position).Magnitude) or 0
                targetName.Text      = "👤 " .. p.DisplayName
                targetName.TextColor3= TXT
                targetDist.Text      = dist .. " st"
            end
        else
            targetName.Text       = "nenhum"
            targetName.TextColor3 = SUB
            targetDist.Text       = ""
        end
    end
end)

print("🍕 Pizza Hub v3.5 carregado! F1 = GUI | SHIFT = Aimbot | ESP ativo")
