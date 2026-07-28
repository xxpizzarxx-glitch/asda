-- ╔══════════════════════════════════════════════════════════╗
-- ║              🍕  PIZZA HUB  🍕                           ║
-- ║      Wallhack · Aimbot · Triggerbot · GUI Moderna        ║
-- ║      Otimizado para máxima performance / mínimo FPS loss ║
-- ╚══════════════════════════════════════════════════════════╝
-- Cole no executor (Synapse X, KRNL, Fluxus, Delta...)
-- F1 = abrir/fechar hub | F2 = toggle aimbot rápido

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

local LP     = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ══════════════════════════════════════════════════════════════
--  CONFIG CENTRAL  (edite aqui)
-- ══════════════════════════════════════════════════════════════
local CFG = {
    -- Wallhack
    WH_Enabled       = true,
    WH_EnemyColor    = Color3.fromRGB(255, 60,  60),
    WH_AllyColor     = Color3.fromRGB(60,  220, 100),
    WH_FillTrans     = 0.5,
    WH_OutlineTrans  = 0.0,

    -- Aimbot
    AB_Enabled       = true,
    AB_Key           = Enum.KeyCode.LeftShift,
    AB_FOV           = 120,          -- graus (dot product)
    AB_Stiffness     = 14,           -- spring rigidez
    AB_Damping       = 6,            -- spring amortecimento
    AB_Predict       = 0.10,         -- segundos de predição
    AB_AimPart       = "Head",       -- "Head" | "HumanoidRootPart"
    AB_ShowFOV       = true,
    AB_TeamCheck     = true,
    AB_HealthCheck   = true,
    AB_VisCheck      = true,

    -- Triggerbot
    TB_Enabled       = false,
    TB_Threshold     = 18,           -- pixels do centro
    TB_Delay         = 0.10,         -- segundos entre tiros
    TB_TeamCheck     = true,

    -- GUI
    GUI_AccentA      = Color3.fromRGB(255, 140,  0),   -- laranja pizza
    GUI_AccentB      = Color3.fromRGB(220,  40, 40),   -- vermelho pizza
    GUI_BG           = Color3.fromRGB(10,   10, 14),
    GUI_Card         = Color3.fromRGB(18,   18, 25),
    GUI_Panel        = Color3.fromRGB(24,   24, 34),
    GUI_Text         = Color3.fromRGB(230, 230, 240),
    GUI_Sub          = Color3.fromRGB(130, 130, 150),
    GUI_Toggle       = Color3.fromRGB(30,   30, 46),
    GUI_Green        = Color3.fromRGB(60,  220, 100),
    GUI_Red          = Color3.fromRGB(255,  60,  60),
}

-- ══════════════════════════════════════════════════════════════
--  HELPERS / PERF
-- ══════════════════════════════════════════════════════════════
local V2   = Vector2.new
local V3   = Vector3.new
local CF   = CFrame.new
local cos  = math.cos
local rad  = math.rad
local huge = math.huge
local floor= math.floor
local clamp= math.clamp

-- Cache de personagens (evita GetPlayers() a cada frame)
local charCache   = {}     -- [player] = {char, root, head, hum, vel, highlight}
local CACHE_DIRTY = true

local function RebuildCache()
    for p, data in pairs(charCache) do
        if not data.char or not data.char.Parent then
            charCache[p] = nil
        end
    end
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP then
            local c  = p.Character
            local r  = c and c:FindFirstChild("HumanoidRootPart")
            local h  = c and c:FindFirstChild("Head")
            local hm = c and c:FindFirstChildOfClass("Humanoid")
            if c and r and h and hm then
                charCache[p] = charCache[p] or {}
                local d = charCache[p]
                d.char = c; d.root = r; d.head = h; d.hum = hm
                d.vel  = r.AssemblyLinearVelocity     -- atualizado barato
                d.player = p
            end
        end
    end
    CACHE_DIRTY = false
end

local function IsEnemy(p)
    if not CFG.AB_TeamCheck then return true end
    return p.Team ~= LP.Team
end

local function IsAlive(data)
    return data.hum and data.hum.Health > 0
end

-- Converte ângulo FOV (graus) em dot-product mínimo
local function FOVtoDot(fov)
    return cos(rad(fov * 0.5))
end

-- Direção câmera → alvo normalizada
local function DirTo(worldPos)
    return (worldPos - Camera.CFrame.Position).Unit
end

-- Checa se posição está dentro do FOV configurado
local function InFOV(worldPos, minDot)
    local look = Camera.CFrame.LookVector
    local dir  = DirTo(worldPos)
    return look:Dot(dir) >= minDot
end

-- Screen center
local function ScreenCenter()
    local vp = Camera.ViewportSize
    return V2(vp.X * 0.5, vp.Y * 0.5)
end

-- WorldToViewport seguro (retorna nil se atrás da câmera)
local function W2V(pos)
    local sp, vis = Camera:WorldToViewportPoint(pos)
    if sp.Z > 0 then return V2(sp.X, sp.Y), vis end
    return nil, false
end

-- ══════════════════════════════════════════════════════════════
--  WALLHACK  (Highlight nativo — zero Drawing, zero FPS loss)
-- ══════════════════════════════════════════════════════════════
local function ApplyHighlight(data)
    local p = data.player
    local isEnemy = IsEnemy(p)
    local hl = data.char:FindFirstChildOfClass("SelectionHighlight")
           or  data.highlight

    if not hl or not hl.Parent then
        hl = Instance.new("SelectionHighlight")
        hl.Name      = "PizzaESP"
        hl.Adornee   = data.char
        hl.Parent    = data.char
        data.highlight = hl
    end

    hl.Enabled           = CFG.WH_Enabled
    hl.FillColor         = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor
    hl.OutlineColor      = isEnemy and CFG.WH_EnemyColor or CFG.WH_AllyColor
    hl.FillTransparency  = CFG.WH_FillTrans
    hl.OutlineTransparency = CFG.WH_OutlineTrans
    hl.DepthMode         = Enum.HighlightDepthMode.AlwaysOnTop
end

local function RemoveHighlight(data)
    if data and data.highlight then
        pcall(function() data.highlight:Destroy() end)
        data.highlight = nil
    end
end

local function RefreshHighlights()
    for _, data in pairs(charCache) do
        if data.char then
            if CFG.WH_Enabled then
                ApplyHighlight(data)
            else
                if data.highlight then
                    data.highlight.Enabled = false
                end
            end
        end
    end
end

-- ══════════════════════════════════════════════════════════════
--  AIMBOT  (spring-damper + predição)
-- ══════════════════════════════════════════════════════════════
-- Estado do spring (velocidade angular acumulada)
local springVel = V3(0, 0, 0)

local lockedTarget = nil   -- data entry ou nil
local lockedLabel  = nil   -- atualizado pelo GUI

local function GetAimPart(data)
    return CFG.AB_AimPart == "Head" and data.head or data.root
end

local function GetBestTarget()
    if not CFG.AB_Enabled then return nil end
    local minDot   = FOVtoDot(CFG.AB_FOV)
    local best     = nil
    local bestDot  = -huge

    for p, data in pairs(charCache) do
        if IsEnemy(p) then
            if not (CFG.AB_HealthCheck and not IsAlive(data)) then
                local part = GetAimPart(data)
                if part then
                    local pred = part.Position + (data.vel * CFG.AB_Predict)
                    if CFG.AB_VisCheck then
                        local sp, vis = W2V(pred)
                        if not (sp and vis) then continue end
                    end
                    local dir  = DirTo(pred)
                    local look = Camera.CFrame.LookVector
                    local dot  = look:Dot(dir)
                    if dot >= minDot and dot > bestDot then
                        bestDot = dot
                        best    = data
                    end
                end
            end
        end
    end
    return best
end

-- Aplica spring-damper na direção da câmera (sem teleportar)
local function SpringStep(dt, data)
    local part = GetAimPart(data)
    if not part then return end

    local pred     = part.Position + (data.vel * CFG.AB_Predict)
    local wantDir  = DirTo(pred)
    local curDir   = Camera.CFrame.LookVector

    -- Erro angular como vetor 3D
    local err      = wantDir - curDir

    -- Integração spring (semi-implícita Euler)
    local k        = CFG.AB_Stiffness
    local d        = CFG.AB_Damping
    local acc      = err * k - springVel * d
    springVel      = springVel + acc * dt
    local newDir   = (curDir + springVel * dt).Unit

    local camPos   = Camera.CFrame.Position
    Camera.CFrame  = CFrame.lookAt(camPos, camPos + newDir, V3(0, 1, 0))
end

-- ══════════════════════════════════════════════════════════════
--  TRIGGERBOT
-- ══════════════════════════════════════════════════════════════
local tbCooldown = 0

local function TriggerTick(dt)
    tbCooldown = tbCooldown - dt
    if not CFG.TB_Enabled or tbCooldown > 0 then return end

    local center = ScreenCenter()
    for p, data in pairs(charCache) do
        if not (CFG.TB_TeamCheck and not IsEnemy(p)) then
            if IsAlive(data) then
                local sp, vis = W2V(data.head.Position)
                if sp and vis then
                    local dist = (sp - center).Magnitude
                    if dist <= CFG.TB_Threshold then
                        -- Simula clique (mouse1click via VirtualUser)
                        local vus = game:GetService("VirtualUser")
                        vus:Button1Down(center, Camera.CFrame)
                        task.delay(0.06, function()
                            vus:Button1Up(center, Camera.CFrame)
                        end)
                        tbCooldown = CFG.TB_Delay
                        break
                    end
                end
            end
        end
    end
end

-- ══════════════════════════════════════════════════════════════
--  FOV CIRCLE  (Drawing — só recria quando muda)
-- ══════════════════════════════════════════════════════════════
local fovCircle = Drawing.new("Circle")
fovCircle.Visible   = false
fovCircle.Color     = CFG.GUI_AccentA
fovCircle.Thickness = 1
fovCircle.NumSides  = 64
fovCircle.Filled    = false
fovCircle.ZIndex    = 10

local fovLastR = 0
local function UpdateFOVCircle()
    local show = CFG.AB_Enabled and CFG.AB_ShowFOV
    fovCircle.Visible = show
    if show then
        local vp = Camera.ViewportSize
        fovCircle.Position = V2(vp.X * 0.5, vp.Y * 0.5)
        -- Converte FOV de graus para raio em pixels (aproximação)
        local r = floor(vp.Y * CFG.AB_FOV / Camera.FieldOfView)
        if r ~= fovLastR then
            fovCircle.Radius = r
            fovLastR = r
        end
    end
end

-- Indicador de alvo travado
local lockLine = Drawing.new("Line")
lockLine.Visible   = false
lockLine.Color     = CFG.GUI_AccentA
lockLine.Thickness = 1.5
lockLine.ZIndex    = 11

local function UpdateLockIndicator(data)
    if not data then
        lockLine.Visible = false
        return
    end
    local part = GetAimPart(data)
    if not part then lockLine.Visible = false; return end
    local sp, vis = W2V(part.Position)
    if sp and vis then
        local center = ScreenCenter()
        lockLine.Visible = true
        lockLine.From    = center
        lockLine.To      = sp
    else
        lockLine.Visible = false
    end
end

-- ══════════════════════════════════════════════════════════════
--  GUI — PIZZA HUB
-- ══════════════════════════════════════════════════════════════
local SG = Instance.new("ScreenGui")
SG.Name              = "PizzaHub"
SG.ResetOnSpawn      = false
SG.ZIndexBehavior    = Enum.ZIndexBehavior.Sibling
SG.IgnoreGuiInset    = true
SG.Parent            = game:GetService("CoreGui")

local function New(cls, props, parent)
    local i = Instance.new(cls)
    for k, v in pairs(props) do i[k] = v end
    if parent then i.Parent = parent end
    return i
end

local function Corner(r, p) return New("UICorner", {CornerRadius = UDim.new(0, r)}, p) end
local function Stroke(c, t, p) return New("UIStroke", {Color=c, Thickness=t}, p) end
local function Pad(l, r, t, b, p)
    return New("UIPadding",{PaddingLeft=UDim.new(0,l),PaddingRight=UDim.new(0,r),
                             PaddingTop=UDim.new(0,t),PaddingBottom=UDim.new(0,b)}, p)
end
local function List(pad, p)
    return New("UIListLayout",{Padding=UDim.new(0,pad),SortOrder=Enum.SortOrder.LayoutOrder}, p)
end

-- Frame principal
local Main = New("Frame", {
    Size             = UDim2.new(0, 370, 0, 500),
    Position         = UDim2.new(0, 50, 0, 50),
    BackgroundColor3 = CFG.GUI_BG,
    BorderSizePixel  = 0,
    Active           = true,
    Draggable        = true,
    ClipsDescendants = true,
}, SG)
Corner(14, Main)
Stroke(CFG.GUI_AccentA, 1.2, Main)

-- Barra de gradiente topo
local TopBar = New("Frame", {
    Size             = UDim2.new(1, 0, 0, 4),
    BackgroundColor3 = CFG.GUI_AccentA,
    BorderSizePixel  = 0,
    ZIndex           = 5,
}, Main)
Corner(14, TopBar)
New("UIGradient", {
    Color    = ColorSequence.new(CFG.GUI_AccentA, CFG.GUI_AccentB),
    Rotation = 0,
}, TopBar)

-- Header
local Header = New("Frame", {
    Size             = UDim2.new(1, 0, 0, 56),
    Position         = UDim2.new(0, 0, 0, 4),
    BackgroundTransparency = 1,
}, Main)

New("TextLabel", {
    Size             = UDim2.new(0, 180, 1, 0),
    Position         = UDim2.new(0, 14, 0, 0),
    BackgroundTransparency = 1,
    Text             = "🍕 PIZZA HUB",
    TextColor3       = CFG.GUI_Text,
    Font             = Enum.Font.GothamBold,
    TextSize         = 18,
    TextXAlignment   = Enum.TextXAlignment.Left,
}, Header)

New("TextLabel", {
    Size             = UDim2.new(0, 160, 0, 16),
    Position         = UDim2.new(0, 14, 1, -18),
    BackgroundTransparency = 1,
    Text             = "v2.0 · F1 fechar · F2 aimbot",
    TextColor3       = CFG.GUI_Sub,
    Font             = Enum.Font.Gotham,
    TextSize         = 11,
    TextXAlignment   = Enum.TextXAlignment.Left,
}, Header)

-- Status dot
local statusDot = New("Frame", {
    Size             = UDim2.new(0, 9, 0, 9),
    Position         = UDim2.new(1, -46, 0.5, -4),
    BackgroundColor3 = CFG.GUI_Green,
    BorderSizePixel  = 0,
}, Header)
Corner(99, statusDot)

New("TextLabel", {
    Size             = UDim2.new(0, 38, 1, 0),
    Position         = UDim2.new(1, -86, 0, 0),
    BackgroundTransparency = 1,
    Text             = "Online",
    TextColor3       = CFG.GUI_Green,
    Font             = Enum.Font.GothamBold,
    TextSize         = 12,
    TextXAlignment   = Enum.TextXAlignment.Right,
}, Header)

-- Divider
New("Frame", {
    Size             = UDim2.new(1, -28, 0, 1),
    Position         = UDim2.new(0, 14, 0, 60),
    BackgroundColor3 = Color3.fromRGB(38, 38, 55),
    BorderSizePixel  = 0,
}, Main)

-- Tabs
local TabBar = New("Frame", {
    Size             = UDim2.new(1, 0, 0, 38),
    Position         = UDim2.new(0, 0, 0, 64),
    BackgroundTransparency = 1,
}, Main)
Pad(10, 10, 0, 0, TabBar)
List(6, TabBar)
New("UIListLayout", {
    Padding          = UDim.new(0, 6),
    FillDirection    = Enum.FillDirection.Horizontal,
    SortOrder        = Enum.SortOrder.LayoutOrder,
}, TabBar)

local tabPages = {}
local tabBtns  = {}
local activeTab = nil

local function MakeTab(name, emoji, order)
    local btn = New("TextButton", {
        Size             = UDim2.new(0, 90, 0, 30),
        BackgroundColor3 = CFG.GUI_Card,
        Text             = emoji .. " " .. name,
        TextColor3       = CFG.GUI_Sub,
        Font             = Enum.Font.GothamBold,
        TextSize         = 12,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, TabBar)
    Corner(7, btn)

    local page = New("ScrollingFrame", {
        Size             = UDim2.new(1, 0, 1, -108),
        Position         = UDim2.new(0, 0, 0, 104),
        BackgroundTransparency = 1,
        ScrollBarThickness = 2,
        ScrollBarImageColor3 = CFG.GUI_AccentA,
        BorderSizePixel  = 0,
        Visible          = false,
        CanvasSize       = UDim2.new(0, 0, 0, 0),
        AutomaticCanvasSize = Enum.AutomaticSize.Y,
    }, Main)
    Pad(12, 12, 6, 10, page)
    List(8, page)

    tabPages[name] = page
    tabBtns[name]  = btn
    return page
end

local function SelectTab(name)
    for n, p in pairs(tabPages) do
        p.Visible = (n == name)
        local b = tabBtns[n]
        TweenService:Create(b, TweenInfo.new(0.18), {
            BackgroundColor3 = n == name and CFG.GUI_AccentA or CFG.GUI_Card,
            TextColor3       = n == name and Color3.new(1,1,1) or CFG.GUI_Sub,
        }):Play()
    end
    activeTab = name
end

local pgMain  = MakeTab("Main",    "🍕", 1)
local pgAim   = MakeTab("Aimbot",  "🎯", 2)
local pgExtra = MakeTab("Config",  "⚙",  3)

for name, btn in pairs(tabBtns) do
    btn.MouseButton1Click:Connect(function() SelectTab(name) end)
end
SelectTab("Main")

-- ── Componentes reutilizáveis ──────────────────────────────────

local function SectionLabel(txt, parent, order)
    local f = New("Frame", {
        Size             = UDim2.new(1, 0, 0, 24),
        BackgroundTransparency = 1,
        LayoutOrder      = order,
    }, parent)
    New("TextLabel", {
        Size             = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text             = txt,
        TextColor3       = CFG.GUI_AccentA,
        Font             = Enum.Font.GothamBold,
        TextSize         = 11,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, f)
    New("Frame", {
        Size             = UDim2.new(1, 0, 0, 1),
        Position         = UDim2.new(0, 0, 1, -1),
        BackgroundColor3 = CFG.GUI_AccentA,
        BackgroundTransparency = 0.7,
        BorderSizePixel  = 0,
    }, f)
    return f
end

local function MakeToggle(parent, label, initVal, onChange, order)
    local row = New("Frame", {
        Size             = UDim2.new(1, 0, 0, 44),
        BackgroundColor3 = CFG.GUI_Card,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, parent)
    Corner(8, row)

    New("TextLabel", {
        Size             = UDim2.new(1, -62, 1, 0),
        Position         = UDim2.new(0, 12, 0, 0),
        BackgroundTransparency = 1,
        Text             = label,
        TextColor3       = CFG.GUI_Text,
        Font             = Enum.Font.Gotham,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, row)

    local pill = New("Frame", {
        Size             = UDim2.new(0, 42, 0, 22),
        Position         = UDim2.new(1, -52, 0.5, -11),
        BackgroundColor3 = initVal and CFG.GUI_AccentA or CFG.GUI_Toggle,
        BorderSizePixel  = 0,
    }, row)
    Corner(99, pill)

    local knob = New("Frame", {
        Size             = UDim2.new(0, 17, 0, 17),
        Position         = initVal and UDim2.new(1,-20,0.5,-8) or UDim2.new(0,3,0.5,-8),
        BackgroundColor3 = Color3.new(1,1,1),
        BorderSizePixel  = 0,
    }, pill)
    Corner(99, knob)

    local active = initVal
    local ti     = TweenInfo.new(0.18, Enum.EasingStyle.Quad)

    local btn = New("TextButton", {
        Size             = UDim2.new(1, 0, 1, 0),
        BackgroundTransparency = 1,
        Text             = "",
        ZIndex           = 5,
    }, row)
    btn.MouseButton1Click:Connect(function()
        active = not active
        TweenService:Create(pill, ti, {BackgroundColor3 = active and CFG.GUI_AccentA or CFG.GUI_Toggle}):Play()
        TweenService:Create(knob, ti, {Position = active and UDim2.new(1,-20,0.5,-8) or UDim2.new(0,3,0.5,-8)}):Play()
        onChange(active)
    end)
    -- Retorna setter externo
    return function(v)
        active = v
        TweenService:Create(pill, ti, {BackgroundColor3 = active and CFG.GUI_AccentA or CFG.GUI_Toggle}):Play()
        TweenService:Create(knob, ti, {Position = active and UDim2.new(1,-20,0.5,-8) or UDim2.new(0,3,0.5,-8)}):Play()
    end
end

local function MakeSlider(parent, label, min, max, cur, fmt, onChange, order)
    local card = New("Frame", {
        Size             = UDim2.new(1, 0, 0, 58),
        BackgroundColor3 = CFG.GUI_Card,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, parent)
    Corner(8, card)

    New("TextLabel", {
        Size             = UDim2.new(1, -70, 0, 22),
        Position         = UDim2.new(0, 12, 0, 6),
        BackgroundTransparency = 1,
        Text             = label,
        TextColor3       = CFG.GUI_Text,
        Font             = Enum.Font.Gotham,
        TextSize          = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, card)

    local valLbl = New("TextLabel", {
        Size             = UDim2.new(0, 60, 0, 22),
        Position         = UDim2.new(1, -70, 0, 6),
        BackgroundTransparency = 1,
        Text             = string.format(fmt, cur),
        TextColor3       = CFG.GUI_AccentA,
        Font             = Enum.Font.GothamBold,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Right,
    }, card)

    local track = New("Frame", {
        Size             = UDim2.new(1, -24, 0, 4),
        Position         = UDim2.new(0, 12, 0, 38),
        BackgroundColor3 = CFG.GUI_Toggle,
        BorderSizePixel  = 0,
    }, card)
    Corner(99, track)

    local t0    = (cur - min) / (max - min)
    local fill  = New("Frame", {
        Size             = UDim2.new(t0, 0, 1, 0),
        BackgroundColor3 = CFG.GUI_AccentA,
        BorderSizePixel  = 0,
    }, track)
    Corner(99, fill)

    local dragging = false
    track.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = true end
    end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if dragging and i.UserInputType == Enum.UserInputType.MouseMovement then
            local ax = track.AbsolutePosition.X
            local aw = track.AbsoluteSize.X
            local t  = clamp((i.Position.X - ax) / aw, 0, 1)
            local v  = min + t * (max - min)
            -- arredonda baseado no tipo
            if fmt:find("d") then v = floor(v + 0.5) end
            fill.Size    = UDim2.new(t, 0, 1, 0)
            valLbl.Text  = string.format(fmt, v)
            onChange(v)
        end
    end)
end

local function MakeInfoCard(parent, order)
    local card = New("Frame", {
        Size             = UDim2.new(1, 0, 0, 56),
        BackgroundColor3 = CFG.GUI_Card,
        BorderSizePixel  = 0,
        LayoutOrder      = order,
    }, parent)
    Corner(8, card)
    Stroke(CFG.GUI_AccentA, 0.8, card)
    New("UIStroke", {Color=CFG.GUI_AccentA, Thickness=0.8, Transparency=0.5}, card)

    New("TextLabel", {
        Size             = UDim2.new(0, 80, 0, 18),
        Position         = UDim2.new(0, 12, 0, 8),
        BackgroundTransparency = 1,
        Text             = "🎯 ALVO",
        TextColor3       = CFG.GUI_AccentA,
        Font             = Enum.Font.GothamBold,
        TextSize         = 11,
    }, card)

    local nameLbl = New("TextLabel", {
        Size             = UDim2.new(1, -20, 0, 18),
        Position         = UDim2.new(0, 12, 0, 26),
        BackgroundTransparency = 1,
        Text             = "— Nenhum alvo —",
        TextColor3       = CFG.GUI_Sub,
        Font             = Enum.Font.Gotham,
        TextSize         = 13,
        TextXAlignment   = Enum.TextXAlignment.Left,
    }, card)

    local distLbl = New("TextLabel", {
        Size             = UDim2.new(0, 90, 0, 18),
        Position         = UDim2.new(1, -100, 0, 8),
        BackgroundTransparency = 1,
        Text             = "",
        TextColor3       = CFG.GUI_Green,
        Font             = Enum.Font.GothamBold,
        TextSize         = 12,
        TextXAlignment   = Enum.TextXAlignment.Right,
    }, card)

    return nameLbl, distLbl
end

-- ══════════════════ PÁGINA MAIN ════════════════════════════════
SectionLabel("◈  WALLHACK", pgMain, 1)

local setWH = MakeToggle(pgMain, "👁  Highlight (ver através das paredes)", CFG.WH_Enabled, function(v)
    CFG.WH_Enabled = v
    RefreshHighlights()
end, 2)

MakeToggle(pgMain, "⚔  Team Check (não marcar aliados)", CFG.AB_TeamCheck, function(v)
    CFG.AB_TeamCheck = v
    CFG.TB_TeamCheck = v
    CACHE_DIRTY = true
end, 3)

SectionLabel("◈  AIMBOT", pgMain, 4)

local setAB = MakeToggle(pgMain, "🤖  Aimbot (segurar SHIFT)", CFG.AB_Enabled, function(v)
    CFG.AB_Enabled = v
    UpdateFOVCircle()
    if not v then springVel = V3(0,0,0); lockedTarget = nil end
end, 5)

local setTB = MakeToggle(pgMain, "🔫  Triggerbot (auto-atirar no alvo)", CFG.TB_Enabled, function(v)
    CFG.TB_Enabled = v
end, 6)

SectionLabel("◈  STATUS DO ALVO", pgMain, 7)
lockedLabel = MakeInfoCard(pgMain, 8) -- retorna nameLbl, distLbl
local targetNameLbl, targetDistLbl = lockedLabel, nil

-- (recria para capturar os dois retornos corretamente)
do
    -- remove o anterior
    for _, c in ipairs(pgMain:GetChildren()) do
        if c.LayoutOrder == 8 then c:Destroy() end
    end
    targetNameLbl, targetDistLbl = MakeInfoCard(pgMain, 8)
end

-- ══════════════════ PÁGINA AIMBOT ══════════════════════════════
SectionLabel("◈  MIRA", pgAim, 1)

MakeSlider(pgAim, "FOV do Aimbot (graus)", 10, 360, CFG.AB_FOV, "%d°", function(v)
    CFG.AB_FOV = v; UpdateFOVCircle()
end, 2)

MakeSlider(pgAim, "Rigidez (Stiffness)", 1, 30, CFG.AB_Stiffness, "%.0f", function(v)
    CFG.AB_Stiffness = v
end, 3)

MakeSlider(pgAim, "Amortecimento (Damping)", 1, 20, CFG.AB_Damping, "%.0f", function(v)
    CFG.AB_Damping = v
end, 4)

MakeSlider(pgAim, "Predição de movimento (s)", 0, 0.5, CFG.AB_Predict, "%.2fs", function(v)
    CFG.AB_Predict = v
end, 5)

MakeSlider(pgAim, "Delay Triggerbot (s)", 0.02, 0.5, CFG.TB_Delay, "%.2fs", function(v)
    CFG.TB_Delay = v
end, 6)

MakeSlider(pgAim, "Threshold Triggerbot (px)", 5, 80, CFG.TB_Threshold, "%dpx", function(v)
    CFG.TB_Threshold = v
end, 7)

SectionLabel("◈  VISUALIZAÇÃO", pgAim, 8)

MakeToggle(pgAim, "⭕  Mostrar círculo de FOV", CFG.AB_ShowFOV, function(v)
    CFG.AB_ShowFOV = v; UpdateFOVCircle()
end, 9)

MakeToggle(pgAim, "📍  Linha para alvo travado", true, function(v)
    lockLine.Visible = v and (lockedTarget ~= nil)
end, 10)

-- ══════════════════ PÁGINA CONFIG ══════════════════════════════
SectionLabel("◈  CHECAGENS", pgExtra, 1)

MakeToggle(pgExtra, "❤  Health Check (não mirar em mortos)", CFG.AB_HealthCheck, function(v)
    CFG.AB_HealthCheck = v
end, 2)

MakeToggle(pgExtra, "🔍  Visibility Check (WorldToViewport)", CFG.AB_VisCheck, function(v)
    CFG.AB_VisCheck = v
end, 3)

SectionLabel("◈  MIRA EM", pgExtra, 4)

local headBtn = New("TextButton", {
    Size             = UDim2.new(0.5, -5, 0, 36),
    BackgroundColor3 = CFG.AB_AimPart == "Head" and CFG.GUI_AccentA or CFG.GUI_Card,
    Text             = "🗡 Cabeça",
    TextColor3       = Color3.new(1,1,1),
    Font             = Enum.Font.GothamBold,
    TextSize         = 13,
    BorderSizePixel  = 0,
    LayoutOrder      = 5,
}, pgExtra)
Corner(8, headBtn)

local bodyBtn = New("TextButton", {
    Size             = UDim2.new(0.5, -5, 0, 36),
    BackgroundColor3 = CFG.AB_AimPart ~= "Head" and CFG.GUI_AccentA or CFG.GUI_Card,
    Text             = "🛡 Tronco",
    TextColor3       = Color3.new(1,1,1),
    Font             = Enum.Font.GothamBold,
    TextSize         = 13,
    BorderSizePixel  = 0,
    LayoutOrder      = 6,
}, pgExtra)
Corner(8, bodyBtn)

local hrz = New("Frame", {
    Size             = UDim2.new(1, 0, 0, 36),
    BackgroundTransparency = 1,
    LayoutOrder      = 5,
}, pgExtra)
New("UIListLayout", {
    FillDirection = Enum.FillDirection.Horizontal,
    Padding       = UDim.new(0, 8),
}, hrz)
headBtn.Parent = hrz
bodyBtn.Parent = hrz

local ti18 = TweenInfo.new(0.18)
headBtn.MouseButton1Click:Connect(function()
    CFG.AB_AimPart = "Head"
    TweenService:Create(headBtn, ti18, {BackgroundColor3 = CFG.GUI_AccentA}):Play()
    TweenService:Create(bodyBtn, ti18, {BackgroundColor3 = CFG.GUI_Card}):Play()
end)
bodyBtn.MouseButton1Click:Connect(function()
    CFG.AB_AimPart = "HumanoidRootPart"
    TweenService:Create(bodyBtn, ti18, {BackgroundColor3 = CFG.GUI_AccentA}):Play()
    TweenService:Create(headBtn, ti18, {BackgroundColor3 = CFG.GUI_Card}):Play()
end)

SectionLabel("◈  CORES DO WALLHACK", pgExtra, 7)

-- Info cores
New("TextLabel", {
    Size             = UDim2.new(1, 0, 0, 36),
    BackgroundTransparency = 1,
    Text             = "Inimigos: Vermelho  |  Aliados: Verde\n(configurável no topo do script)",
    TextColor3       = CFG.GUI_Sub,
    Font             = Enum.Font.Gotham,
    TextSize         = 12,
    TextXAlignment   = Enum.TextXAlignment.Left,
    TextWrapped      = true,
    LayoutOrder      = 8,
}, pgExtra)

-- ══════════════════════════════════════════════════════════════
--  EVENTOS DE PLAYER
-- ══════════════════════════════════════════════════════════════
local function OnCharAdded(p, char)
    task.wait(0.3)   -- espera partes carregarem
    CACHE_DIRTY = true
    -- Aplica highlight após spawn
    char.ChildAdded:Connect(function()
        CACHE_DIRTY = true
    end)
end

Players.PlayerAdded:Connect(function(p)
    CACHE_DIRTY = true
    p.CharacterAdded:Connect(function(c) OnCharAdded(p, c) end)
    if p.Character then OnCharAdded(p, p.Character) end
end)

Players.PlayerRemoving:Connect(function(p)
    RemoveHighlight(charCache[p])
    charCache[p] = nil
end)

for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LP then
        CACHE_DIRTY = true
        p.CharacterAdded:Connect(function(c) OnCharAdded(p, c) end)
        if p.Character then OnCharAdded(p, p.Character) end
    end
end

-- ══════════════════════════════════════════════════════════════
--  ATALHOS DE TECLADO
-- ══════════════════════════════════════════════════════════════
local aimHeld = false

UserInputService.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.F1 then
        Main.Visible = not Main.Visible
    elseif i.KeyCode == Enum.KeyCode.F2 then
        CFG.AB_Enabled = not CFG.AB_Enabled
        setAB(CFG.AB_Enabled)
        UpdateFOVCircle()
    elseif i.KeyCode == CFG.AB_Key then
        aimHeld = true
    end
end)
UserInputService.InputEnded:Connect(function(i)
    if i.KeyCode == CFG.AB_Key then
        aimHeld = false
        springVel = V3(0, 0, 0)
    end
end)

-- ══════════════════════════════════════════════════════════════
--  LOOP PRINCIPAL — otimizado por tick alternado
-- ══════════════════════════════════════════════════════════════
local frameCount   = 0
local CACHE_EVERY  = 12    -- reconstrói cache a cada N frames (~5x/s a 60fps)
local WH_EVERY     = 20    -- atualiza highlights a cada N frames
local GUI_EVERY    = 15    -- atualiza info de alvo na GUI

RunService.RenderStepped:Connect(function(dt)
    frameCount = frameCount + 1

    -- Rebuild cache (barato, mas não a cada frame)
    if CACHE_DIRTY or frameCount % CACHE_EVERY == 0 then
        RebuildCache()
        -- Aplica highlights em novos personagens
        for _, data in pairs(charCache) do
            if data.highlight == nil or not data.highlight.Parent then
                ApplyHighlight(data)
            end
        end
    end

    -- Wallhack: atualiza Enabled (toggle) periodicamente
    if frameCount % WH_EVERY == 0 then
        for _, data in pairs(charCache) do
            if data.highlight then
                data.highlight.Enabled = CFG.WH_Enabled
            end
        end
    end

    -- Aimbot
    if CFG.AB_Enabled and aimHeld then
        if frameCount % 2 == 0 then   -- roda a cada 2 frames (30fps) — suficiente
            lockedTarget = GetBestTarget()
        end
        if lockedTarget then
            SpringStep(dt, lockedTarget)
        end
    else
        lockedTarget = nil
    end

    -- FOV circle
    UpdateFOVCircle()

    -- Lock indicator
    UpdateLockIndicator(lockedTarget)

    -- Triggerbot
    TriggerTick(dt)

    -- Atualiza GUI de alvo
    if frameCount % GUI_EVERY == 0 then
        if lockedTarget and lockedTarget.char and lockedTarget.char.Parent then
            local dist = floor(GetDistance and
                (lockedTarget.root.Position - (LP.Character and
                LP.Character:FindFirstChild("HumanoidRootPart") and
                LP.Character.HumanoidRootPart.Position or V3())) .Magnitude or 0)
            targetNameLbl.Text  = "👤 " .. lockedTarget.player.DisplayName
            targetNameLbl.TextColor3 = CFG.GUI_Text
            targetDistLbl.Text  = dist .. " st"
        else
            targetNameLbl.Text  = "— Nenhum alvo —"
            targetNameLbl.TextColor3 = CFG.GUI_Sub
            targetDistLbl.Text  = ""
        end
    end
end)

-- ══════════════════════════════════════════════════════════════
print("🍕 Pizza Hub carregado! F1 = GUI  |  F2 = Aimbot  |  SHIFT = Mirar")
