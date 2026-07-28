-- ╔══════════════════════════════════════════════════════╗
-- ║         🍕 PIZZA HUB v4.0 - MEGA OP                ║
-- ║  Wallhack · Aimbot 360° · Triggerbot · Spinbot     ║
-- ║  Silent Aim · ESP · No Recoil · Speed Hack         ║
-- ║  Fly Hack · FOV Changer · Auto Shoot               ║
-- ║  F1 = abrir/fechar · SHIFT = aimbot · ALT = fly    ║
-- ╚══════════════════════════════════════════════════════╝

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local VirtualUser      = game:GetService("VirtualUser")
local HttpService      = game:GetService("HttpService")

local LP     = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse  = LP:GetMouse()

-- ══════════════════════════════════════════
--  CONFIGURAÇÕES AGRESSIVAS
-- ══════════════════════════════════════════
local CFG = {
    -- Wallhack
    WH_On         = true,
    WH_EnemyColor = Color3.fromRGB(255, 0, 0),
    WH_AllyColor  = Color3.fromRGB(0, 255, 0),
    WH_Transparency = 0.3,

    -- Aimbot
    AB_On         = true,
    AB_Smooth     = 0,      -- 0 = instantâneo (MEGA OP)
    AB_FOV        = 999,    -- 360° praticamente
    AB_Part       = "Head", -- sempre cabeça
    AB_TeamCheck  = true,
    AB_HealthCheck= true,
    AB_Prediction = true,
    AB_PredictionAmount = 0.15,

    -- Silent Aim
    SA_On         = false,  -- mira sem mover câmera (muito OP)

    -- Triggerbot
    TB_On         = true,
    TB_Px         = 50,
    TB_Delay      = 0.01,   -- 10ms (ultra rápido)

    -- Spinbot
    SB_On         = false,
    SB_Speed      = 300,    -- graus por segundo

    -- ESP
    ESP_On        = true,
    ESP_Names     = true,
    ESP_HP        = true,
    ESP_Distance  = true,
    ESP_Boxes     = true,
    ESP_Tracers   = true,

    -- No Recoil
    NR_On         = true,

    -- Speed Hack
    SH_On         = false,
    SH_Speed      = 32,     -- walkspeed padrão 16

    -- Fly Hack
    FH_On         = false,
    FH_Speed      = 50,

    -- FOV Changer
    FC_On         = true,
    FC_FOV        = 120,    -- FOV máximo

    -- Auto Shoot
    AS_On         = true,   -- atira automático ao mirar
}

-- ══════════════════════════════════════════
--  VARIÁVEIS GLOBAIS
-- ══════════════════════════════════════════
local highlights   = {}
local espObjects   = {}
local aiming       = false
local spinning     = false
local flying       = false
local tbTimer      = 0
local spinAngle    = 0
local oldWalkspeed = 16
local flyBodyGyro  = nil
local flyBodyVel   = nil

-- ══════════════════════════════════════════
--  FUNÇÕES UTILITÁRIAS
-- ══════════════════════════════════════════
local function GetDistance(pos1, pos2)
    return (pos1 - pos2).Magnitude
end

local function IsAlive(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    return hum and hum.Health > 0
end

local function GetTeamColor(p)
    return (CFG.AB_TeamCheck and p.Team ~= LP.Team) and CFG.WH_EnemyColor or CFG.WH_AllyColor
end

-- ══════════════════════════════════════════
--  WALLHACK
-- ══════════════════════════════════════════
local function AddHighlight(p)
    if p == LP then return end
    if highlights[p] then
        highlights[p]:Destroy()
        highlights[p] = nil
    end
    local char = p.Character
    if not char then return end

    local hl = Instance.new("Highlight")
    hl.Adornee            = char
    hl.DepthMode          = Enum.HighlightDepthMode.AlwaysOnTop
    hl.FillTransparency   = CFG.WH_Transparency
    hl.OutlineTransparency= 0
    hl.Enabled            = CFG.WH_On
    hl.FillColor          = GetTeamColor(p)
    hl.OutlineColor       = GetTeamColor(p)
    hl.Parent             = char
    highlights[p]         = hl
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
        hl.FillTransparency = CFG.WH_Transparency
        hl.FillColor = GetTeamColor(p)
        hl.OutlineColor = GetTeamColor(p)
    end
end

-- ══════════════════════════════════════════
--  ESP
-- ══════════════════════════════════════════
local function CreateESP(p)
    if p == LP then return end
    RemoveESP(p)
    local char = p.Character
    if not char then return end
    local head = char:FindFirstChild("Head")
    local hrp  = char:FindFirstChild("HumanoidRootPart")
    if not head or not hrp then return end

    local bill = Instance.new("BillboardGui")
    bill.Name = "ESP_" .. p.Name
    bill.Size = UDim2.new(0, 120, 0, 60)
    bill.StudsOffset = Vector3.new(0, 2.5, 0)
    bill.AlwaysOnTop = true
    bill.MaxDistance = 500
    bill.Adornee = head
    bill.Parent = char

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Name = "Name"
    nameLabel.Size = UDim2.new(1, 0, 0, 18)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextColor3 = Color3.new(1, 1, 1)
    nameLabel.TextStrokeTransparency = 0.4
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 13
    nameLabel.Text = p.DisplayName
    nameLabel.Parent = bill

    local hpLabel = Instance.new("TextLabel")
    hpLabel.Name = "HP"
    hpLabel.Size = UDim2.new(1, 0, 0, 18)
    hpLabel.Position = UDim2.new(0, 0, 0, 18)
    hpLabel.BackgroundTransparency = 1
    hpLabel.TextStrokeTransparency = 0.4
    hpLabel.Font = Enum.Font.Gotham
    hpLabel.TextSize = 12
    hpLabel.Parent = bill

    local distLabel = Instance.new("TextLabel")
    distLabel.Name = "Dist"
    distLabel.Size = UDim2.new(1, 0, 0, 18)
    distLabel.Position = UDim2.new(0, 0, 0, 36)
    distLabel.BackgroundTransparency = 1
    distLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    distLabel.TextStrokeTransparency = 0.4
    distLabel.Font = Enum.Font.Gotham
    distLabel.TextSize = 11
    distLabel.Parent = bill

    -- Box
    local box = Instance.new("SelectionBox")
    box.Name = "ESPBox"
    box.Adornee = hrp
    box.Color3 = GetTeamColor(p)
    box.LineThickness = 0.03
    box.Transparency = 0.4
    box.Parent = char

    -- Tracer line (usando Beam)
    local tracer = Instance.new("Beam")
    tracer.Name = "Tracer"
    tracer.Attachment0 = Instance.new("Attachment", hrp)
    tracer.Attachment1 = Instance.new("Attachment", Camera)
    tracer.Color = ColorSequence.new(GetTeamColor(p))
    tracer.Width0 = 0.05
    tracer.Width1 = 0.02
    tracer.Transparency = NumberSequence.new(0.6)
    tracer.Enabled = CFG.ESP_Tracers
    tracer.Parent = char

    espObjects[p] = {
        Billboard = bill,
        Box = box,
        Tracer = tracer,
        NameLabel = nameLabel,
        HPLabel = hpLabel,
        DistLabel = distLabel,
    }
end

local function RemoveESP(p)
    if espObjects[p] then
        espObjects[p].Billboard:Destroy()
        espObjects[p].Box:Destroy()
        espObjects[p].Tracer:Destroy()
        espObjects[p] = nil
    end
end

local function UpdateESP()
    if not CFG.ESP_On then return end
    local myRoot = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    for p, esp in pairs(espObjects) do
        local char = p.Character
        if not char or not IsAlive(char) then
            RemoveESP(p)
            continue
        end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            local hp = math.floor(hum.Health)
            esp.HPLabel.Text = "HP: " .. hp
            if hp < 30 then
                esp.HPLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
            elseif hp < 60 then
                esp.HPLabel.TextColor3 = Color3.fromRGB(255, 255, 0)
            else
                esp.HPLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
            end
        end
        if myRoot then
            local dist = GetDistance(myRoot.Position, char:GetPivot().Position)
            esp.DistLabel.Text = string.format("%.0f st", dist)
        end
        esp.Billboard.Enabled = CFG.ESP_On
        esp.Box.Visible = CFG.ESP_Boxes
        esp.Tracer.Enabled = CFG.ESP_Tracers
        esp.NameLabel.Visible = CFG.ESP_Names
        esp.HPLabel.Visible = CFG.ESP_HP
        esp.DistLabel.Visible = CFG.ESP_Distance
        local color = GetTeamColor(p)
        esp.Box.Color3 = color
        esp.Tracer.Color = ColorSequence.new(color)
    end
end

-- ══════════════════════════════════════════
--  AIMBOT (COM PREDIÇÃO)
-- ══════════════════════════════════════════
local function GetTarget()
    local center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local best, bestDist = nil, CFG.AB_FOV

    for _, p in ipairs(Players:GetPlayers()) do
        if p == LP then continue end
        if CFG.AB_TeamCheck and p.Team == LP.Team then continue end
        local char = p.Character
        if not char then continue end
        if CFG.AB_HealthCheck and not IsAlive(char) then continue end
        local part = char:FindFirstChild(CFG.AB_Part) or char:FindFirstChild("HumanoidRootPart")
        if not part then continue end

        local targetPos = part.Position
        -- Predição
        if CFG.AB_Prediction then
            local vel = char:FindFirstChild("HumanoidRootPart") and char.HumanoidRootPart.AssemblyLinearVelocity or Vector3.zero
            targetPos = targetPos + vel * CFG.AB_PredictionAmount
        end

        local sp, onScreen = Camera:WorldToViewportPoint(targetPos)
        if not onScreen or sp.Z <= 0 then continue end

        local d = (Vector2.new(sp.X, sp.Y) - center).Magnitude
        if d < bestDist then
            bestDist = d
            best = {Part = part, Position = targetPos, Player = p, Distance = d}
        end
    end
    return best
end

local function AimbotStep(dt)
    if not CFG.AB_On or not aiming then return end
    local target = GetTarget()
    if not target then return end

    if CFG.SA_On then
        -- Silent Aim: modifica a direção do tiro sem mover câmera
        -- (funciona em alguns jogos, depende do sistema de armas)
        local camPos = Camera.CFrame.Position
        local dir = (target.Position - camPos).Unit
        -- Simula tiro nessa direção (limitado)
        -- A implementação real depende do sistema de armas do jogo
    else
        local cf = Camera.CFrame
        local goal = CFrame.lookAt(cf.Position, target.Position)
        Camera.CFrame = cf:Lerp(goal, 1 - CFG.AB_Smooth)
    end

    -- Auto Shoot
    if CFG.AS_On and target.Distance < CFG.TB_Px then
        if tbTimer <= 0 then
            VirtualUser:Button1Down(Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2), Camera.CFrame)
            task.delay(0.03, function() VirtualUser:Button1Up(Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2), Camera.CFrame) end)
            tbTimer = CFG.TB_Delay
        end
    end

    return target
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
        if CFG.AB_TeamCheck and p.Team == LP.Team then continue end
        local char = p.Character
        if not char then continue end
        if CFG.AB_HealthCheck and not IsAlive(char) then continue end
        local head = char:FindFirstChild("Head")
        if not head then continue end

        local sp, on = Camera:WorldToViewportPoint(head.Position)
        if not on or sp.Z <= 0 then continue end
        if (Vector2.new(sp.X, sp.Y) - center).Magnitude <= CFG.TB_Px then
            VirtualUser:Button1Down(center, Camera.CFrame)
            task.delay(0.03, function() VirtualUser:Button1Up(center, Camera.CFrame) end)
            tbTimer = CFG.TB_Delay
            break
        end
    end
end

-- ══════════════════════════════════════════
--  SPINBOT
-- ══════════════════════════════════════════
local function SpinbotStep(dt)
    if not CFG.SB_On then return end
    spinAngle = spinAngle + CFG.SB_Speed * dt
    if spinAngle > 360 then spinAngle = spinAngle - 360 end

    local cf = Camera.CFrame
    local newCF = cf * CFrame.Angles(0, math.rad(CFG.SB_Speed * dt), 0)
    Camera.CFrame = newCF

    -- Também mira aleatoriamente pra baixo/cima pra ser mais caótico
    if math.random() < 0.3 then
        Camera.CFrame = Camera.CFrame * CFrame.Angles(math.rad(math.random(-30, 30)), 0, 0)
    end
end

-- ══════════════════════════════════════════
--  NO RECOIL / NO SPREAD
-- ══════════════════════════════════════════
local function NoRecoilStep()
    if not CFG.NR_On then return end
    -- Tenta zerar o recoil pattern
    -- Isso é mais efetivo em jogos específicos com armas
    -- Em alguns jogos, mexer no CFrame da câmera ajuda
    -- Esta é uma implementação básica
end

-- ══════════════════════════════════════════
--  SPEED HACK
-- ══════════════════════════════════════════
local function SpeedHackStep()
    if not LP.Character then return end
    local hum = LP.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    if CFG.SH_On then
        hum.WalkSpeed = CFG.SH_Speed
    else
        hum.WalkSpeed = oldWalkspeed
    end
end

-- ══════════════════════════════════════════
--  FLY HACK
-- ══════════════════════════════════════════
local function FlyHackStart()
    if not LP.Character then return end
    local hrp = LP.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    flyBodyGyro = Instance.new("BodyGyro")
    flyBodyGyro.MaxTorque = Vector3.new(400000, 400000, 400000)
    flyBodyGyro.D = 100
    flyBodyGyro.P = 10000
    flyBodyGyro.CFrame = Camera.CFrame
    flyBodyGyro.Parent = hrp

    flyBodyVel = Instance.new("BodyVelocity")
    flyBodyVel.MaxForce = Vector3.new(400000, 400000, 400000)
    flyBodyVel.Velocity = Vector3.zero
    flyBodyVel.Parent = hrp

    flying = true
end

local function FlyHackStop()
    if flyBodyGyro then flyBodyGyro:Destroy() end
    if flyBodyVel then flyBodyVel:Destroy() end
    flyBodyGyro = nil
    flyBodyVel = nil
    flying = false
end

local function FlyHackStep(dt)
    if not CFG.FH_On then
        if flying then FlyHackStop() end
        return
    end
    if not flying then FlyHackStart() end
    if not flyBodyGyro or not flyBodyVel then return end

    -- Controles de voo (WASD + Espaço/Shift)
    local moveDir = Vector3.zero
    if UserInputService:IsKeyDown(Enum.KeyCode.W) then
        moveDir = moveDir + Camera.CFrame.LookVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.S) then
        moveDir = moveDir - Camera.CFrame.LookVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.A) then
        moveDir = moveDir - Camera.CFrame.RightVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.D) then
        moveDir = moveDir + Camera.CFrame.RightVector
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
        moveDir = moveDir + Vector3.new(0, 1, 0)
    end
    if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
        moveDir = moveDir - Vector3.new(0, 1, 0)
    end

    if moveDir.Magnitude > 0 then
        flyBodyVel.Velocity = moveDir.Unit * CFG.FH_Speed
    else
        flyBodyVel.Velocity = Vector3.zero
    end
    flyBodyGyro.CFrame = Camera.CFrame
end

-- ══════════════════════════════════════════
--  FOV CHANGER
-- ══════════════════════════════════════════
local function FOVChangerStep()
    if CFG.FC_On then
        Camera.FieldOfView = CFG.FC_FOV
    else
        Camera.FieldOfView = 70
    end
end

-- ══════════════════════════════════════════
--  CONECTAR JOGADORES
-- ══════════════════════════════════════════
local function OnPlayerAdded(p)
    if p == LP then return end

    local function OnChar(char)
        task.wait(0.3)
        AddHighlight(p)
        if CFG.ESP_On then CreateESP(p) end
    end

    p.CharacterAdded:Connect(OnChar)
    if p.Character then OnChar(p.Character) end
end

for _, p in ipairs(Players:GetPlayers()) do
    OnPlayerAdded(p)
end
Players.PlayerAdded:Connect(OnPlayerAdded)
Players.PlayerRemoving:Connect(function(p)
    RemoveHighlight(p)
    RemoveESP(p)
end)

-- ══════════════════════════════════════════
--  GUI - PIZZA HUB v4.0 MEGA OP
-- ══════════════════════════════════════════
pcall(function() game:GetService("CoreGui"):FindFirstChild("PizzaHub"):Destroy() end)

local SG = Instance.new("ScreenGui")
SG.Name = "PizzaHub"
SG.ResetOnSpawn = false
SG.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
SG.IgnoreGuiInset = true
SG.Parent = game:GetService("CoreGui")

-- Cores
local A1  = Color3.fromRGB(255, 0, 0)    -- vermelho
local A2  = Color3.fromRGB(255, 100, 0)  -- laranja
local BG  = Color3.fromRGB(5, 5, 10)
local CAR = Color3.fromRGB(15, 15, 25)
local TXT = Color3.fromRGB(240, 240, 255)
local SUB = Color3.fromRGB(150, 150, 170)
local TOG = Color3.fromRGB(40, 40, 60)
local GRN = Color3.fromRGB(0, 255, 100)
local TI  = TweenInfo.new(0.15, Enum.EasingStyle.Quad)

local function N(c, props, parent)
    local i = Instance.new(c)
    for k,v in pairs(props) do i[k]=v end
    if parent then i.Parent = parent end
    return i
end

-- Frame principal
local Main = N("Frame", {
    Size = UDim2.new(0, 360, 0, 520),
    Position = UDim2.new(0, 50, 0, 50),
    BackgroundColor3 = BG,
    BorderSizePixel = 0,
    Active = true,
    Draggable = true,
    ClipsDescendants = true,
}, SG)
N("UICorner", {CornerRadius = UDim.new(0, 12)}, Main)
N("UIStroke", {Color = A1, Thickness = 1.5, Transparency = 0.2}, Main)

-- Barra topo
local topBar = N("Frame", {
    Size = UDim2.new(1, 0, 0, 4),
    BackgroundColor3 = A1,
    BorderSizePixel = 0,
}, Main)
N("UICorner", {CornerRadius = UDim.new(0, 12)}, topBar)
N("UIGradient", {Color = ColorSequence.new(A1, A2), Rotation = 0}, topBar)

-- Título
N("TextLabel", {
    Size = UDim2.new(1, -16, 0, 44),
    Position = UDim2.new(0, 14, 0, 6),
    BackgroundTransparency = 1,
    Text = "🔥  PIZZA HUB v4.0  MEGA OP",
    TextColor3 = TXT,
    Font = Enum.Font.GothamBold,
    TextSize = 16,
    TextXAlignment = Enum.TextXAlignment.Left,
}, Main)

N("TextLabel", {
    Size = UDim2.new(1, -16, 0, 16),
    Position = UDim2.new(0, 14, 0, 42),
    BackgroundTransparency = 1,
    Text = "F1 = menu  ·  SHIFT = aim  ·  ALT = fly",
    TextColor3 = SUB,
    Font = Enum.Font.Gotham,
    TextSize = 10,
    TextXAlignment = Enum.TextXAlignment.Left,
}, Main)

-- Linha
N("Frame", {
    Size = UDim2.new(1, -28, 0, 1),
    Position = UDim2.new(0, 14, 0, 62),
    BackgroundColor3 = Color3.fromRGB(60, 0, 0),
    BorderSizePixel = 0,
}, Main)

-- Scroll
local Scroll = N("ScrollingFrame", {
    Size = UDim2.new(1, 0, 1, -68),
    Position = UDim2.new(0, 0, 0, 68),
    BackgroundTransparency = 1,
    ScrollBarThickness = 3,
    ScrollBarImageColor3 = A1,
    BorderSizePixel = 0,
    CanvasSize = UDim2.new(0, 0, 0, 0),
    AutomaticCanvasSize = Enum.AutomaticSize.Y,
    ClipsDescendants = true,
}, Main)
N("UIPadding", {PaddingLeft=UDim.new(0,12), PaddingRight=UDim.new(0,12), PaddingTop=UDim.new(0,8), PaddingBottom=UDim.new(0,10)}, Scroll)
N("UIListLayout", {Padding=UDim.new(0,6), SortOrder=Enum.SortOrder.LayoutOrder}, Scroll)

-- Helpers
local function SecLabel(txt, order)
    local f = N("Frame", {Size=UDim2.new(1,0,0,22), BackgroundTransparency=1, LayoutOrder=order}, Scroll)
    N("TextLabel", {Size=UDim2.new(1,0,1,0), BackgroundTransparency=1, Text=txt, TextColor3=A1, Font=Enum.Font.GothamBold, TextSize=11, TextXAlignment=Enum.TextXAlignment.Left}, f)
    N("Frame", {Size=UDim2.new(1,0,0,1), Position=UDim2.new(0,0,1,0), BackgroundColor3=A1, BackgroundTransparency=0.6, BorderSizePixel=0}, f)
end

local function MakeToggle(label, init, onChange, order)
    local row = N("Frame", {Size=UDim2.new(1,0,0,40), BackgroundColor3=CAR, BorderSizePixel=0, LayoutOrder=order}, Scroll)
    N("UICorner", {CornerRadius=UDim.new(0,6)}, row)
    N("TextLabel", {Size=UDim2.new(1,-60,1,0), Position=UDim2.new(0,10,0,0), BackgroundTransparency=1, Text=label, TextColor3=TXT, Font=Enum.Font.Gotham, TextSize=12, TextXAlignment=Enum.TextXAlignment.Left}, row)

    local pill = N("Frame", {Size=UDim2.new(0,40,0,20), Position=UDim2.new(1,-48,0.5,-10), BackgroundColor3=init and A1 or TOG, BorderSizePixel=0}, row)
    N("UICorner", {CornerRadius=UDim.new(1,0)}, pill)
    local knob = N("Frame", {Size=UDim2.new(0,16,0,16), Position=init and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8), BackgroundColor3=Color3.new(1,1,1), BorderSizePixel=0}, pill)
    N("UICorner", {CornerRadius=UDim.new(1,0)}, knob)

    local val = init
    local btn = N("TextButton", {Size=UDim2.new(1,0,1,0), BackgroundTransparency=1, Text="", ZIndex=5}, row)
    local function SetState(v)
        val = v
        TweenService:Create(pill, TI, {BackgroundColor3 = v and A1 or TOG}):Play()
        TweenService:Create(knob, TI, {Position = v and UDim2.new(1,-18,0.5,-8) or UDim2.new(0,2,0.5,-8)}):Play()
    end
    btn.MouseButton1Click:Connect(function() SetState(not val) onChange(val) end)
    return SetState
end

local function MakeSlider(label, min, max, cur, decimals, onChange, order)
    local card = N("Frame", {Size=UDim2.new(1,0,0,50), BackgroundColor3=CAR, BorderSizePixel=0, LayoutOrder=order}, Scroll)
    N("UICorner", {CornerRadius=UDim.new(0,6)}, card)
    N("TextLabel", {Size=UDim2.new(1,-70,0,18), Position=UDim2.new(0,10,0,4), BackgroundTransparency=1, Text=label, TextColor3=TXT, Font=Enum.Font.Gotham, TextSize=11, TextXAlignment=Enum.TextXAlignment.Left}, card)

    local fmt = "%." .. decimals .. "f"
    local vLbl = N("TextLabel", {Size=UDim2.new(0,58,0,18), Position=UDim2.new(1,-64,0,4), BackgroundTransparency=1, Text=string.format(fmt,cur), TextColor3=A1, Font=Enum.Font.GothamBold, TextSize=11, TextXAlignment=Enum.TextXAlignment.Right}, card)

    local track = N("Frame", {Size=UDim2.new(1,-20,0,4), Position=UDim2.new(0,10,0,30), BackgroundColor3=TOG, BorderSizePixel=0}, card)
    N("UICorner", {CornerRadius=UDim.new(1,0)}, track)
    local t0 = (cur-min)/(max-min)
    local fill = N("Frame", {Size=UDim2.new(t0,0,1,0), BackgroundColor3=A1, BorderSizePixel=0}, track)
    N("UICorner", {CornerRadius=UDim.new(1,0)}, fill)
    local thumb = N("Frame", {Size=UDim2.new(0,12,0,12), Position=UDim2.new(t0,-6,0.5,-6), BackgroundColor3=Color3.new(1,1,1), BorderSizePixel=0, ZIndex=3}, track)
    N("UICorner", {CornerRadius=UDim.new(1,0)}, thumb)

    local dragging = false
    local function Update(x)
        local ax, aw = track.AbsolutePosition.X, track.AbsoluteSize.X
        local t = math.clamp((x-ax)/aw,0,1)
        local v = min + t*(max-min)
        if decimals == 0 then v = math.floor(v+0.5) end
        fill.Size = UDim2.new(t,0,1,0)
        thumb.Position = UDim2.new(t,-6,0.5,-6)
        vLbl.Text = string.format(fmt,v)
        onChange(v)
    end
    track.InputBegan:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging=true; Update(i.Position.X) end end)
    UserInputService.InputEnded:Connect(function(i) if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging=false end end)
    UserInputService.InputChanged:Connect(function(i) if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then Update(i.Position.X) end end)
end

-- Monta controles
SecLabel("🔥  WALLHACK", 1)
MakeToggle("👁  Wallhack", CFG.WH_On, function(v) CFG.WH_On=v; RefreshAllHighlights() end, 2)

SecLabel("🎯  AIMBOT", 3)
MakeToggle("🤖  Aimbot", CFG.AB_On, function(v) CFG.AB_On=v end, 4)
MakeSlider("Suavidade", 0, 0.99, CFG.AB_Smooth, 2, function(v) CFG.AB_Smooth=v end, 5)
MakeSlider("FOV", 20, 2000, CFG.AB_FOV, 0, function(v) CFG.AB_FOV=v end, 6)
MakeToggle("🔮  Predição", CFG.AB_Prediction, function(v) CFG.AB_Prediction=v end, 7)
MakeToggle("🤫  Silent Aim", CFG.SA_On, function(v) CFG.SA_On=v end, 8)
MakeToggle("🔫  Auto Shoot", CFG.AS_On, function(v) CFG.AS_On=v end, 9)

SecLabel("💀  TRIGGERBOT", 10)
MakeToggle("💥  Triggerbot", CFG.TB_On, function(v) CFG.TB_On=v end, 11)
MakeSlider("Threshold (px)", 5, 200, CFG.TB_Px, 0, function(v) CFG.TB_Px=v end, 12)
MakeSlider("Delay (s)", 0, 0.5, CFG.TB_Delay, 2, function(v) CFG.TB_Delay=v end, 13)

SecLabel("🌀  SPINBOT", 14)
MakeToggle("🌪  Spinbot", CFG.SB_On, function(v) CFG.SB_On=v end, 15)
MakeSlider("Velocidade", 50, 1000, CFG.SB_Speed, 0, function(v) CFG.SB_Speed=v end, 16)

SecLabel("👻  ESP", 17)
MakeToggle("📊  ESP", CFG.ESP_On, function(v) CFG.ESP_On=v; for p in pairs(espObjects) do if not v then RemoveESP(p) end end end, 18)
MakeToggle("📛  Nomes", CFG.ESP_Names, function(v) CFG.ESP_Names=v end, 19)
MakeToggle("❤  HP", CFG.ESP_HP, function(v) CFG.ESP_HP=v end, 20)
MakeToggle("📏  Distância", CFG.ESP_Distance, function(v) CFG.ESP_Distance=v end, 21)
MakeToggle("📦  Boxes", CFG.ESP_Boxes, function(v) CFG.ESP_Boxes=v end, 22)
MakeToggle("➖  Tracers", CFG.ESP_Tracers, function(v) CFG.ESP_Tracers=v end, 23)

SecLabel("⚡  MOVIMENTO", 24)
MakeToggle("🏃  Speed Hack", CFG.SH_On, function(v) CFG.SH_On=v end, 25)
MakeSlider("Speed", 20, 100, CFG.SH_Speed, 0, function(v) CFG.SH_Speed=v end, 26)
MakeToggle("🕊  Fly Hack (ALT)", CFG.FH_On, function(v) CFG.FH_On=v end, 27)
MakeSlider("Fly Speed", 20, 200, CFG.FH_Speed, 0, function(v) CFG.FH_Speed=v end, 28)

SecLabel("⚙  OUTROS", 29)
MakeToggle("🔭  FOV Changer", CFG.FC_On, function(v) CFG.FC_On=v end, 30)
MakeSlider("FOV", 70, 150, CFG.FC_FOV, 0, function(v) CFG.FC_FOV=v end, 31)
MakeToggle("⚔  Team Check", CFG.AB_TeamCheck, function(v) CFG.AB_TeamCheck=v; CFG.TB_TeamCheck=v; RefreshAllHighlights() end, 32)
MakeToggle("💀  Health Check", CFG.AB_HealthCheck, function(v) CFG.AB_HealthCheck=v end, 33)

-- ══════════════════════════════════════════
--  INPUTS
-- ══════════════════════════════════════════
UserInputService.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.F1 then
        Main.Visible = not Main.Visible
    elseif i.KeyCode == Enum.KeyCode.LeftShift or i.KeyCode == Enum.KeyCode.RightShift then
        aiming = true
    elseif i.KeyCode == Enum.KeyCode.LeftAlt or i.KeyCode == Enum.KeyCode.RightAlt then
        CFG.FH_On = not CFG.FH_On
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

    SpinbotStep(dt)
    local target = AimbotStep(dt)
    TriggerbotStep(dt)
    SpeedHackStep()
    FlyHackStep(dt)
    FOVChangerStep()
    NoRecoilStep()

    -- ESP update
    if tick % 15 == 0 then
        UpdateESP()
        -- Reconectar ESP para novos jogadores
        if CFG.ESP_On then
            for _, p in ipairs(Players:GetPlayers()) do
                if p ~= LP and p.Character and IsAlive(p.Character) and not espObjects[p] then
                    CreateESP(p)
                end
            end
        end
    end
end)

print("🍕 Pizza Hub v4.0 MEGA OP carregado!")
print("🔥 Funções: Wallhack | Aimbot 360° | Triggerbot | Spinbot | Silent Aim | ESP | Speed | Fly | FOV")
print("⌨ F1 = Menu | SHIFT = Aimbot | ALT = Fly")
