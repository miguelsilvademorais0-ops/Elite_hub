--[[
    ═══════════════════════════════════════════════════════════
    ⚡ ELITE HUB | Steal An Egg
    Versão: 2.0.0
    Executor: Delta
    Autor: Elite Hub Dev
    ═══════════════════════════════════════════════════════════
    
    COMO USAR:
    1. Cole este script no seu executor (Delta)
    2. Ou hospede em GitHub/Pastebin e use:
       loadstring(game:HttpGet("SUA_URL"))()
    3. Keybind padrão: RightShift
    ═══════════════════════════════════════════════════════════
]]

--// ============ SERVIÇOS ============
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")
local VirtualUser = game:GetService("VirtualUser")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local StarterGui = game:GetService("StarterGui")
local Lighting = game:GetService("Lighting")

local LP = Players.LocalPlayer
local Camera = workspace.CurrentCamera

--// ============ CONFIG ============
local Config = {
    ESP_Ovos = false,
    ESP_Jogadores = false,
    ESP_Raridade = false,
    ESP_Valor = false,
    ESP_FiltroRaridade = "Todos",
    ESP_FiltroValorMin = 0,
    ESP_Distancia = 500,
    
    Velocidade = 16,
    Pulo = 50,
    AntiRagdoll = false,
    InfiniteJump = false,
    
    AutoHatch = false,
    AutoEquip = false,
    AutoFavorite = false,
    HatchFiltroRaridade = "Todos",
    HatchFiltroValorMin = 0,
    
    AutoSell = false,
    SellFiltroRaridade = "Comum",
    SellFiltroValorMin = 0,
    BlacklistOvos = {},
    BlacklistPets = {},
    
    AutoCollect = false,
    AutoBuy = false,
    AutoProgression = false,
    PrioridadeObjetivo = "Mais Próximo",
    
    BossTarget = "Automático",
    BossPrioridade = "HP Baixo",
    BossReroll = false,
    AutoRecompensa = false,
    
    MostrarFPS = true,
    MostrarPing = true,
    Otimizacao = false,
    AntiAFK = true,
    
    Keybind = "RightShift",
    Notificacoes = true,
    
    AutoSteal = false,
    StealRange = 30,
    AutoFlee = false,
    AutoStealFiltroRaridade = "Todos",
}

local CONFIG_FILE = "elite_hub_config.json"

--// ============ RARIDADES ============
local RARIDADES = {
    ["Comum"] = 1, ["Incomum"] = 2, ["Raro"] = 3, ["Épico"] = 4,
    ["Lendário"] = 5, ["Mítico"] = 6, ["Secreto"] = 7, ["Divino"] = 8,
    ["Common"] = 1, ["Uncommon"] = 2, ["Rare"] = 3, ["Epic"] = 4,
    ["Legendary"] = 5, ["Mythic"] = 6, ["Secret"] = 7, ["Divine"] = 8,
}

local RARIDADE_CORES = {
    ["Comum"] = Color3.fromRGB(180, 180, 180),
    ["Incomum"] = Color3.fromRGB(80, 200, 80),
    ["Raro"] = Color3.fromRGB(80, 130, 255),
    ["Épico"] = Color3.fromRGB(180, 80, 255),
    ["Lendário"] = Color3.fromRGB(255, 180, 50),
    ["Mítico"] = Color3.fromRGB(255, 60, 60),
    ["Secreto"] = Color3.fromRGB(255, 100, 200),
    ["Divino"] = Color3.fromRGB(255, 255, 150),
}

--// ============ REMOTES ============
local Remotes = {
    Hatch = nil, Equip = nil, Favorite = nil,
    Sell = nil, Collect = nil, Buy = nil,
    Steal = nil, Attack = nil, Reroll = nil, Reward = nil,
}

local function EncontrarRemotes()
    local padroes = {
        Hatch = {"hatch", "open", "abrir"},
        Equip = {"equip", "equipar"},
        Favorite = {"favorite", "favoritar"},
        Sell = {"sell", "vender"},
        Collect = {"collect", "coletar", "pickup"},
        Buy = {"buy", "comprar", "purchase"},
        Steal = {"steal", "roubar", "grab", "take"},
        Attack = {"attack", "atacar", "damage", "hit"},
        Reroll = {"reroll", "rerolar", "refresh"},
        Reward = {"reward", "recompensa", "claim"},
    }
    
    local function scan(container)
        if not container then return end
        for _, obj in ipairs(container:GetDescendants()) do
            if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
                local nome = string.lower(obj.Name)
                for tipo, palavras in pairs(padroes) do
                    if not Remotes[tipo] then
                        for _, p in ipairs(palavras) do
                            if string.find(nome, p) then
                                Remotes[tipo] = obj
                                break
                            end
                        end
                    end
                end
            end
        end
    end
    
    pcall(scan, ReplicatedStorage)
    pcall(scan, workspace)
    
    local encontrados = {}
    for k, v in pairs(Remotes) do
        if v then table.insert(encontrados, k) end
    end
    print("⚡ Elite Hub | Remotes:", table.concat(encontrados, ", "))
end

EncontrarRemotes()

--// ============ HELPERS ============
local function Notificar(titulo, texto, dur)
    if not Config.Notificacoes then return end
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = titulo, Text = texto, Duration = dur or 3
        })
    end)
end

local function ObterDadosItem(obj)
    if not obj then return "Comum", 0, "?" end
    local raridade, valor, nome = "Comum", 0, obj.Name
    
    pcall(function()
        if obj:GetAttribute("Rarity") then raridade = tostring(obj:GetAttribute("Rarity")) end
        if obj:GetAttribute("Raridade") then raridade = tostring(obj:GetAttribute("Raridade")) end
        if obj:GetAttribute("Value") then valor = tonumber(obj:GetAttribute("Value")) or 0 end
        if obj:GetAttribute("Price") then valor = tonumber(obj:GetAttribute("Price")) or 0 end
    end)
    
    pcall(function()
        for _, c in ipairs(obj:GetChildren()) do
            local n = string.lower(c.Name)
            if c:IsA("StringValue") and (n == "rarity" or n == "raridade") then
                raridade = c.Value
            elseif (c:IsA("NumberValue") or c:IsA("IntValue")) and 
                (n == "value" or n == "valor" or n == "price" or n == "preco") then
                valor = c.Value
            end
        end
    end)
    
    if raridade == "Comum" then
        for r, _ in pairs(RARIDADES) do
            if string.find(string.lower(obj.Name), string.lower(r)) then
                raridade = r; break
            end
        end
    end
    return raridade, valor, nome
end

local function PassaFiltro(raridade, valor, filtroR, filtroV)
    if filtroR ~= "Todos" and raridade ~= filtroR then return false end
    if valor and valor < (filtroV or 0) then return false end
    return true
end

local function DistanciaPara(obj)
    local char = LP.Character
    if not char then return math.huge end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local p = obj.Position or (obj:IsA("Model") and obj.PrimaryPart and obj.PrimaryPart.Position)
    if not hrp or not p then return math.huge end
    return (hrp.Position - p).Magnitude
end

local function TeleportarPara(pos)
    local char = LP.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if hrp and pos then
        hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
    end
end

--// ============ UI ============
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "EliteHub"
ScreenGui.ResetOnSpawn = false
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = LP:WaitForChild("PlayerGui")

local Main = Instance.new("Frame")
Main.Size = UDim2.new(0, 620, 0, 440)
Main.Position = UDim2.new(0.5, -310, 0.5, -220)
Main.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.Parent = ScreenGui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 10)

local stroke = Instance.new("UIStroke", Main)
stroke.Color = Color3.fromRGB(180, 100, 255)
stroke.Thickness = 2

local TopBar = Instance.new("Frame")
TopBar.Size = UDim2.new(1, 0, 0, 38)
TopBar.BackgroundColor3 = Color3.fromRGB(28, 28, 38)
TopBar.BorderSizePixel = 0
TopBar.Parent = Main
Instance.new("UICorner", TopBar).CornerRadius = UDim.new(0, 10)

local Titulo = Instance.new("TextLabel")
Titulo.Size = UDim2.new(1, -90, 1, 0)
Titulo.Position = UDim2.new(0, 14, 0, 0)
Titulo.BackgroundTransparency = 1
Titulo.Text = "⚡ ELITE HUB | Steal An Egg"
Titulo.TextColor3 = Color3.fromRGB(200, 130, 255)
Titulo.TextSize = 17
Titulo.Font = Enum.Font.GothamBold
Titulo.TextXAlignment = Enum.TextXAlignment.Left
Titulo.Parent = TopBar

local MinimizarBtn = Instance.new("TextButton")
MinimizarBtn.Size = UDim2.new(0, 28, 0, 28)
MinimizarBtn.Position = UDim2.new(1, -64, 0, 4)
MinimizarBtn.BackgroundColor3 = Color3.fromRGB(200, 150, 50)
MinimizarBtn.Text = "–"
MinimizarBtn.TextColor3 = Color3.fromRGB(20, 20, 25)
MinimizarBtn.TextSize = 18
MinimizarBtn.Font = Enum.Font.GothamBold
MinimizarBtn.BorderSizePixel = 0
MinimizarBtn.Parent = TopBar
Instance.new("UICorner", MinimizarBtn).CornerRadius = UDim.new(0, 6)

local FecharBtn = Instance.new("TextButton")
FecharBtn.Size = UDim2.new(0, 28, 0, 28)
FecharBtn.Position = UDim2.new(1, -32, 0, 4)
FecharBtn.BackgroundColor3 = Color3.fromRGB(220, 60, 60)
FecharBtn.Text = "X"
FecharBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
FecharBtn.TextSize = 14
FecharBtn.Font = Enum.Font.GothamBold
FecharBtn.BorderSizePixel = 0
FecharBtn.Parent = TopBar
Instance.new("UICorner", FecharBtn).CornerRadius = UDim.new(0, 6)

local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(0, 145, 1, -48)
TabContainer.Position = UDim2.new(0, 5, 0, 43)
TabContainer.BackgroundTransparency = 1
TabContainer.Parent = Main

local TabScroll = Instance.new("ScrollingFrame")
TabScroll.Size = UDim2.new(1, 0, 1, 0)
TabScroll.BackgroundTransparency = 1
TabScroll.ScrollBarThickness = 3
TabScroll.ScrollBarImageColor3 = Color3.fromRGB(180, 100, 255)
TabScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
TabScroll.Parent = TabContainer

local TabLayout = Instance.new("UIListLayout", TabScroll)
TabLayout.Padding = UDim.new(0, 4)
TabLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    TabScroll.CanvasSize = UDim2.new(0, 0, 0, TabLayout.AbsoluteContentSize.Y + 10)
end)

local ContentFrame = Instance.new("Frame")
ContentFrame.Size = UDim2.new(1, -160, 1, -48)
ContentFrame.Position = UDim2.new(0, 155, 0, 43)
ContentFrame.BackgroundColor3 = Color3.fromRGB(24, 24, 32)
ContentFrame.BorderSizePixel = 0
ContentFrame.Parent = Main
Instance.new("UICorner", ContentFrame).CornerRadius = UDim.new(0, 8)

local ScrollFrame = Instance.new("ScrollingFrame")
ScrollFrame.Size = UDim2.new(1, -10, 1, -10)
ScrollFrame.Position = UDim2.new(0, 5, 0, 5)
ScrollFrame.BackgroundTransparency = 1
ScrollFrame.ScrollBarThickness = 4
ScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(180, 100, 255)
ScrollFrame.Parent = ContentFrame

local ScrollLayout = Instance.new("UIListLayout", ScrollFrame)
ScrollLayout.Padding = UDim.new(0, 6)
ScrollLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    ScrollFrame.CanvasSize = UDim2.new(0, 0, 0, ScrollLayout.AbsoluteContentSize.Y + 10)
end)

--// ============ COMPONENTES ============
local Abas = {}

local function CriarAba(nome, icone, ordem)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 32)
    btn.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
    btn.Text = icone .. " " .. nome
    btn.TextColor3 = Color3.fromRGB(200, 200, 200)
    btn.TextSize = 13
    btn.Font = Enum.Font.GothamMedium
    btn.BorderSizePixel = 0
    btn.LayoutOrder = ordem
    btn.Parent = TabScroll
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 0)
    frame.AutomaticSize = Enum.AutomaticSize.Y
    frame.BackgroundTransparency = 1
    frame.Visible = false
    frame.Parent = ScrollFrame
    
    Abas[nome] = {Botao = btn, Frame = frame}
    
    btn.MouseButton1Click:Connect(function()
        for _, aba in pairs(Abas) do
            aba.Frame.Visible = false
            aba.Botao.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
            aba.Botao.TextColor3 = Color3.fromRGB(200, 200, 200)
        end
        frame.Visible = true
        btn.BackgroundColor3 = Color3.fromRGB(180, 100, 255)
        btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end)
    return frame
end

local function Toggle(parent, texto, padrao, cb)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(1, -10, 0, 26)
    f.BackgroundTransparency = 1
    f.Parent = parent
    
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -50, 1, 0)
    l.BackgroundTransparency = 1
    l.Text = texto
    l.TextColor3 = Color3.fromRGB(220, 220, 220)
    l.TextSize = 13
    l.Font = Enum.Font.Gotham
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = f
    
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, 40, 0, 20)
    b.Position = UDim2.new(1, -40, 0, 3)
    b.BackgroundColor3 = padrao and Color3.fromRGB(100, 220, 120) or Color3.fromRGB(60, 60, 72)
    b.Text = ""
    b.BorderSizePixel = 0
    b.Parent = f
    Instance.new("UICorner", b).CornerRadius = UDim.new(1, 0)
    
    local c = Instance.new("Frame")
    c.Size = UDim2.new(0, 16, 0, 16)
    c.Position = padrao and UDim2.new(1, -18, 0, 2) or UDim2.new(0, 2, 0, 2)
    c.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    c.BorderSizePixel = 0
    c.Parent = b
    Instance.new("UICorner", c).CornerRadius = UDim.new(1, 0)
    
    local estado = padrao
    b.MouseButton1Click:Connect(function()
        estado = not estado
        TweenService:Create(c, TweenInfo.new(0.15), {
            Position = estado and UDim2.new(1, -18, 0, 2) or UDim2.new(0, 2, 0, 2)
        }):Play()
        b.BackgroundColor3 = estado and Color3.fromRGB(100, 220, 120) or Color3.fromRGB(60, 60, 72)
        if cb then cb(estado) end
    end)
    return f
end

local function Slider(parent, texto, min, max, padrao, cb)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(1, -10, 0, 42)
    f.BackgroundTransparency = 1
    f.Parent = parent
    
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, 0, 0, 18)
    l.BackgroundTransparency = 1
    l.Text = texto .. ": " .. padrao
    l.TextColor3 = Color3.fromRGB(220, 220, 220)
    l.TextSize = 13
    l.Font = Enum.Font.Gotham
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = f
    
    local bar = Instance.new("Frame")
    bar.Size = UDim2.new(1, 0, 0, 8)
    bar.Position = UDim2.new(0, 0, 0, 26)
    bar.BackgroundColor3 = Color3.fromRGB(60, 60, 72)
    bar.BorderSizePixel = 0
    bar.Parent = f
    Instance.new("UICorner", bar).CornerRadius = UDim.new(1, 0)
    
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new((padrao - min) / (max - min), 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(180, 100, 255)
    fill.BorderSizePixel = 0
    fill.Parent = bar
    Instance.new("UICorner", fill).CornerRadius = UDim.new(1, 0)
    
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 1, 20)
    btn.Position = UDim2.new(0, 0, 0, -6)
    btn.BackgroundTransparency = 1
    btn.Text = ""
    btn.Parent = bar
    
    local drag = false
    local val = padrao
    
    local function update(x)
        local rel = math.clamp((x - bar.AbsolutePosition.X) / bar.AbsoluteSize.X, 0, 1)
        val = math.floor(min + (max - min) * rel)
        fill.Size = UDim2.new(rel, 0, 1, 0)
        l.Text = texto .. ": " .. val
        if cb then cb(val) end
    end
    
    btn.MouseButton1Down:Connect(function() drag = true end)
    UserInputService.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 then drag = false end
    end)
    UserInputService.InputChanged:Connect(function(i)
        if drag and i.UserInputType == Enum.UserInputType.MouseMovement then
            update(i.Position.X)
        end
    end)
    return f
end

local function Dropdown(parent, texto, opcoes, padrao, cb)
    local f = Instance.new("Frame")
    f.Size = UDim2.new(1, -10, 0, 28)
    f.BackgroundTransparency = 1
    f.Parent = parent
    
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(0.4, 0, 1, 0)
    l.BackgroundTransparency = 1
    l.Text = texto
    l.TextColor3 = Color3.fromRGB(220, 220, 220)
    l.TextSize = 13
    l.Font = Enum.Font.Gotham
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = f
    
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0.55, 0, 0, 24)
    b.Position = UDim2.new(0.45, 0, 0, 2)
    b.BackgroundColor3 = Color3.fromRGB(48, 48, 60)
    b.Text = padrao or opcoes[1]
    b.TextColor3 = Color3.fromRGB(255, 255, 255)
    b.TextSize = 12
    b.Font = Enum.Font.Gotham
    b.BorderSizePixel = 0
    b.Parent = f
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 5)
    
    local lst = Instance.new("Frame")
    lst.Size = UDim2.new(0.55, 0, 0, 0)
    lst.Position = UDim2.new(0.45, 0, 1, 2)
    lst.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
    lst.BorderSizePixel = 0
    lst.Visible = false
    lst.ZIndex = 10
    lst.Parent = f
    Instance.new("UICorner", lst).CornerRadius = UDim.new(0, 5)
    Instance.new("UIListLayout", lst)
    
    for _, op in ipairs(opcoes) do
        local ob = Instance.new("TextButton")
        ob.Size = UDim2.new(1, 0, 0, 22)
        ob.BackgroundColor3 = Color3.fromRGB(38, 38, 50)
        ob.Text = op
        ob.TextColor3 = Color3.fromRGB(220, 220, 220)
        ob.TextSize = 12
        ob.Font = Enum.Font.Gotham
        ob.BorderSizePixel = 0
        ob.ZIndex = 11
        ob.Parent = lst
        
        ob.MouseButton1Click:Connect(function()
            b.Text = op
            lst.Visible = false
            lst.Size = UDim2.new(0.55, 0, 0, 0)
            if cb then cb(op) end
        end)
    end
    
    local aberto = false
    b.MouseButton1Click:Connect(function()
        aberto = not aberto
        lst.Visible = aberto
        lst.Size = aberto and UDim2.new(0.55, 0, 0, #opcoes * 22) or UDim2.new(0.55, 0, 0, 0)
    end)
    return f
end

local function Botao(parent, texto, cb)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, -10, 0, 28)
    b.BackgroundColor3 = Color3.fromRGB(180, 100, 255)
    b.Text = texto
    b.TextColor3 = Color3.fromRGB(255, 255, 255)
    b.TextSize = 13
    b.Font = Enum.Font.GothamBold
    b.BorderSizePixel = 0
    b.Parent = parent
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
    b.MouseButton1Click:Connect(cb)
    return b
end

local function Titulo(parent, texto)
    local l = Instance.new("TextLabel")
    l.Size = UDim2.new(1, -10, 0, 24)
    l.BackgroundTransparency = 1
    l.Text = "─ " .. texto .. " ─"
    l.TextColor3 = Color3.fromRGB(200, 130, 255)
    l.TextSize = 13
    l.Font = Enum.Font.GothamBold
    l.TextXAlignment = Enum.TextXAlignment.Left
    l.Parent = parent
    return l
end

local function Input(parent, placeholder)
    local tb = Instance.new("TextBox")
    tb.Size = UDim2.new(1, -10, 0, 26)
    tb.BackgroundColor3 = Color3.fromRGB(48, 48, 60)
    tb.PlaceholderText = placeholder
    tb.Text = ""
    tb.TextColor3 = Color3.fromRGB(255, 255, 255)
    tb.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
    tb.TextSize = 12
    tb.Font = Enum.Font.Gotham
    tb.BorderSizePixel = 0
    tb.Parent = parent
    Instance.new("UICorner", tb).CornerRadius = UDim.new(0, 5)
    return tb
end

--// ============ ABAS ============

-- ESP
local EspAba = CriarAba("ESP", "👁", 1)
Titulo(EspAba, "Visualização")
Toggle(EspAba, "Mostrar Ovos", false, function(v) Config.ESP_Ovos = v end)
Toggle(EspAba, "Mostrar Jogadores", false, function(v) Config.ESP_Jogadores = v end)
Toggle(EspAba, "Mostrar Raridade", false, function(v) Config.ESP_Raridade = v end)
Toggle(EspAba, "Mostrar Valor", false, function(v) Config.ESP_Valor = v end)
Titulo(EspAba, "Filtros")
Dropdown(EspAba, "Raridade:", {"Todos", "Comum", "Incomum", "Raro", "Épico", "Lendário", "Mítico", "Secreto", "Divino"}, "Todos", function(v) Config.ESP_FiltroRaridade = v end)
Slider(EspAba, "Valor mínimo", 0, 100000, 0, function(v) Config.ESP_FiltroValorMin = v end)
Slider(EspAba, "Distância máx.", 50, 2000, 500, function(v) Config.ESP_Distancia = v end)
Titulo(EspAba, "🎯 Roubo (Steal)")
Toggle(EspAba, "Auto Steal", false, function(v) Config.AutoSteal = v end)
Slider(EspAba, "Alcance", 5, 100, 30, function(v) Config.StealRange = v end)
Toggle(EspAba, "Auto Fugir após roubar", false, function(v) Config.AutoFlee = v end)

-- MOVIMENTO
local MovAba = CriarAba("Movimento", "🏃", 2)
Titulo(MovAba, "Controle")
Slider(MovAba, "Velocidade", 16, 500, 16, function(v)
    Config.Velocidade = v
    local c = LP.Character
    if c and c:FindFirstChildOfClass("Humanoid") then c:FindFirstChildOfClass("Humanoid").WalkSpeed = v end
end)
Slider(MovAba, "Pulo", 50, 500, 50, function(v)
    Config.Pulo = v
    local c = LP.Character
    if c and c:FindFirstChildOfClass("Humanoid") then c:FindFirstChildOfClass("Humanoid").JumpPower = v end
end)
Titulo(MovAba, "Extras")
Toggle(MovAba, "Anti-Ragdoll", false, function(v) Config.AntiRagdoll = v end)
Toggle(MovAba, "Pulo Infinito", false, function(v) Config.InfiniteJump = v end)

-- OVOS
local OvosAba = CriarAba("Ovos", "🥚", 3)
Titulo(OvosAba, "Automação")
Toggle(OvosAba, "Auto Hatch", false, function(v) Config.AutoHatch = v end)
Toggle(OvosAba, "Auto Equip", false, function(v) Config.AutoEquip = v end)
Toggle(OvosAba, "Auto Favorite", false, function(v) Config.AutoFavorite = v end)
Titulo(OvosAba, "Filtros")
Dropdown(OvosAba, "Raridade:", {"Todos", "Comum", "Incomum", "Raro", "Épico", "Lendário", "Mítico", "Secreto", "Divino"}, "Todos", function(v) Config.HatchFiltroRaridade = v end)
Slider(OvosAba, "Valor mínimo", 0, 100000, 0, function(v) Config.HatchFiltroValorMin = v end)

-- INVENTÁRIO
local InvAba = CriarAba("Inventário", "🎒", 4)
Titulo(InvAba, "Venda Automática")
Toggle(InvAba, "Auto Sell", false, function(v) Config.AutoSell = v end)
Dropdown(InvAba, "Vender raridade:", {"Todos", "Comum", "Incomum", "Raro", "Épico", "Lendário"}, "Comum", function(v) Config.SellFiltroRaridade = v end)
Slider(InvAba, "Valor máx.", 0, 100000, 0, function(v) Config.SellFiltroValorMin = v end)

Titulo(InvAba, "Blacklist de Ovos")
local inputOvo = Input(InvAba, "Nome do ovo para bloquear...")
Botao(InvAba, "➕ Adicionar Ovo à Blacklist", function()
    if inputOvo.Text ~= "" then
        table.insert(Config.BlacklistOvos, inputOvo.Text)
        Notificar("Blacklist", inputOvo.Text .. " adicionado", 2)
        inputOvo.Text = ""
    end
end)

Titulo(InvAba, "Blacklist de Pets")
local inputPet = Input(InvAba, "Nome do pet para bloquear...")
Botao(InvAba, "➕ Adicionar Pet à Blacklist", function()
    if inputPet.Text ~= "" then
        table.insert(Config.BlacklistPets, inputPet.Text)
        Notificar("Blacklist", inputPet.Text .. " adicionado", 2)
        inputPet.Text = ""
    end
end)

Botao(InvAba, "🗑 Limpar Blacklists", function()
    Config.BlacklistOvos = {}
    Config.BlacklistPets = {}
    Notificar("Blacklist", "Limpa!", 2)
end)

-- FARM
local FarmAba = CriarAba("Farm", "⛏", 5)
Titulo(FarmAba, "Automação")
Toggle(FarmAba, "Auto Collect", false, function(v) Config.AutoCollect = v end)
Toggle(FarmAba, "Auto Buy", false, function(v) Config.AutoBuy = v end)
Toggle(FarmAba, "Auto Progression", false, function(v) Config.AutoProgression = v end)
Titulo(FarmAba, "Prioridades")
Dropdown(FarmAba, "Objetivo:", {"Mais Próximo", "Mais Valioso", "Mais Raro"}, "Mais Próximo", function(v) Config.PrioridadeObjetivo = v end)

-- RIFT/BOSS
local BossAba = CriarAba("Rift/Boss", "👹", 6)
Titulo(BossAba, "Seleção")
Dropdown(BossAba, "Alvo:", {"Automático", "Mais Próximo", "Mais Fraco", "Mais Forte"}, "Automático", function(v) Config.BossTarget = v end)
Dropdown(BossAba, "Prioridade:", {"HP Baixo", "HP Alto", "Distância"}, "HP Baixo", function(v) Config.BossPrioridade = v end)
Titulo(BossAba, "Ações")
Toggle(BossAba, "Auto Reroll", false, function(v) Config.BossReroll = v end)
Toggle(BossAba, "Auto Recompensa", false, function(v) Config.AutoRecompensa = v end)

-- CONFIG
local CfgAba = CriarAba("Config", "⚙", 7)
Titulo(CfgAba, "Gerenciamento")
Botao(CfgAba, "💾 Salvar Config", function()
    if writefile then
        writefile(CONFIG_FILE, HttpService:JSONEncode(Config))
        Notificar("Elite Hub", "Config salva!", 2)
    else
        Notificar("Elite Hub", "writefile indisponível", 2)
    end
end)
Botao(CfgAba, "📂 Carregar Config", function()
    if isfile and isfile(CONFIG_FILE) then
        local ok, dados = pcall(function() return HttpService:JSONDecode(readfile(CONFIG_FILE)) end)
        if ok and dados then
            for k, v in pairs(dados) do Config[k] = v end
            Notificar("Elite Hub", "Config carregada! Reative os toggles.", 4)
        end
    else
        Notificar("Elite Hub", "Nenhuma config salva", 2)
    end
end)
Botao(CfgAba, "🔄 Resetar Config", function()
    if delfile and isfile(CONFIG_FILE) then delfile(CONFIG_FILE) end
    Notificar("Elite Hub", "Config resetada!", 2)
end)
Titulo(CfgAba, "Interface")
Toggle(CfgAba, "Notificações", true, function(v) Config.Notificacoes = v end)

-- PERFORMANCE
local PerfAba = CriarAba("Performance", "📊", 8)
Titulo(PerfAba, "Monitor")
Toggle(PerfAba, "Mostrar FPS", true, function(v) Config.MostrarFPS = v end)
Toggle(PerfAba, "Mostrar Ping", true, function(v) Config.MostrarPing = v end)
Titulo(PerfAba, "Otimização")
Toggle(PerfAba, "Otimização Gráfica", false, function(v)
    Config.Otimizacao = v
    pcall(function()
        Lighting.GlobalShadows = not v
        Lighting.FogEnd = v and 500 or 100000
        for _, e in ipairs(Lighting:GetChildren()) do
            if e:IsA("PostEffect") or e:IsA("Atmosphere") then
                e.Enabled = not v
            end
        end
    end)
end)
Toggle(PerfAba, "Anti-AFK", true, function(v) Config.AntiAFK = v end)

--// ============ HUD FPS/PING ============
local Hud = Instance.new("Frame")
Hud.Size = UDim2.new(0, 180, 0, 55)
Hud.Position = UDim2.new(1, -190, 0, 10)
Hud.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
Hud.BackgroundTransparency = 0.2
Hud.BorderSizePixel = 0
Hud.Parent = ScreenGui
Instance.new("UICorner", Hud).CornerRadius = UDim.new(0, 8)

local HudStroke = Instance.new("UIStroke", Hud)
HudStroke.Color = Color3.fromRGB(180, 100, 255)
HudStroke.Thickness = 1.5

local FpsLabel = Instance.new("TextLabel")
FpsLabel.Size = UDim2.new(1, 0, 0, 25)
FpsLabel.BackgroundTransparency = 1
FpsLabel.Text = "FPS: 0"
FpsLabel.TextColor3 = Color3.fromRGB(100, 220, 120)
FpsLabel.TextSize = 14
FpsLabel.Font = Enum.Font.GothamBold
FpsLabel.Parent = Hud

local PingLabel = Instance.new("TextLabel")
PingLabel.Size = UDim2.new(1, 0, 0, 25)
PingLabel.Position = UDim2.new(0, 0, 0, 25)
PingLabel.BackgroundTransparency = 1
PingLabel.Text = "Ping: 0 ms"
PingLabel.TextColor3 = Color3.fromRGB(180, 100, 255)
PingLabel.TextSize = 14
PingLabel.Font = Enum.Font.GothamBold
PingLabel.Parent = Hud

--// ============ ESP ============
local function CriarBillboard(part, texto, cor)
    if not part or not part:IsA("BasePart") then return nil end
    local bb = Instance.new("BillboardGui")
    bb.Size = UDim2.new(0, 120, 0, 40)
    bb.StudsOffset = Vector3.new(0, 3, 0)
    bb.AlwaysOnTop = true
    bb.Parent = part
    
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 0.2
    lbl.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
    lbl.Text = texto
    lbl.TextColor3 = cor or Color3.fromRGB(255, 255, 255)
    lbl.TextSize = 12
    lbl.Font = Enum.Font.GothamBold
    lbl.TextStrokeTransparency = 0.5
    lbl.Parent = bb
    Instance.new("UICorner", lbl).CornerRadius = UDim.new(0, 5)
    
    return bb
end

local function LimparESP()
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") or obj:IsA("Model") then
            local b1 = obj:FindFirstChild("EliteEggESP")
            local b2 = obj:FindFirstChild("ElitePlayerESP")
            if b1 then b1:Destroy() end
            if b2 then b2:Destroy() end
        end
    end
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp and hrp:FindFirstChild("ElitePlayerESP") then
                hrp.ElitePlayerESP:Destroy()
            end
        end
    end
end

RunService.RenderStepped:Connect(function()
    if not Config.ESP_Ovos and not Config.ESP_Jogadores then return end
    
    -- ESP Jogadores
    if Config.ESP_Jogadores then
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                if hrp and not hrp:FindFirstChild("ElitePlayerESP") then
                    local dist = DistanciaPara(hrp)
                    if dist <= Config.ESP_Distancia then
                        local nome = p.Name .. "\n[" .. math.floor(dist) .. "m]"
                        local bb = CriarBillboard(hrp, nome, Color3.fromRGB(255, 80, 80))
                        if bb then bb.Name = "ElitePlayerESP" end
                    end
                end
            end
        end
    else
        for _, p in ipairs(Players:GetPlayers()) do
            if p.Character then
                local hrp = p.Character:FindFirstChild("HumanoidRootPart")
                if hrp and hrp:FindFirstChild("ElitePlayerESP") then
                    hrp.ElitePlayerESP:Destroy()
                end
            end
        end
    end
    
    -- ESP Ovos
    if Config.ESP_Ovos then
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("BasePart") and not obj:FindFirstChild("EliteEggESP") then
                local nome = string.lower(obj.Name)
                if string.find(nome, "egg") or string.find(nome, "ovo") then
                    local dist = DistanciaPara(obj)
                    if dist <= Config.ESP_Distancia then
                        local raridade, valor, nomeObj = ObterDadosItem(obj)
                        if PassaFiltro(raridade, valor, Config.ESP_FiltroRaridade, Config.ESP_FiltroValorMin) then
                            local texto = nomeObj
                            if Config.ESP_Raridade then
                                texto = texto .. "\n[" .. raridade .. "]"
                            end
                            if Config.ESP_Valor then
                                texto = texto .. "\n💰 " .. valor
                            end
                            local cor = RARIDADE_CORES[raridade] or Color3.fromRGB(255, 220, 80)
                            local bb = CriarBillboard(obj, texto, cor)
                            if bb then bb.Name = "EliteEggESP" end
                        end
                    end
                end
            end
        end
    else
        for _, obj in ipairs(workspace:GetDescendants()) do
            if obj:IsA("BasePart") and obj:FindFirstChild("EliteEggESP") then
                obj.EliteEggESP:Destroy()
            end
        end
    end
end)

--// ============ MOVIMENTO ============
RunService.Heartbeat:Connect(function()
    local char = LP.Character
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    
    if hum.WalkSpeed ~= Config.Velocidade then
        hum.WalkSpeed = Config.Velocidade
    end
    if hum.UseJumpPower and hum.JumpPower ~= Config.Pulo then
        hum.JumpPower = Config.Pulo
    end
    
    if Config.AntiRagdoll then
        hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
        hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
    end
end)

-- Pulo Infinito
UserInputService.JumpRequest:Connect(function()
    if Config.InfiniteJump then
        local char = LP.Character
        if char then
            local hum = char:FindFirstChildOfClass("Humanoid")
            if hum then
                hum:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end
    end
end)

--// ============ AUTO FUNÇÕES ============

-- Anti-AFK
LP.Idled:Connect(function()
    if Config.AntiAFK then
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new())
    end
end)

-- Auto Hatch
task.spawn(function()
    while task.wait(0.5) do
        if Config.AutoHatch then
            pcall(function()
                -- Tenta via RemoteEvent primeiro
                if Remotes.Hatch then
                    if Remotes.Hatch:IsA("RemoteEvent") then
                        Remotes.Hatch:FireServer()
                    else
                        Remotes.Hatch:InvokeServer()
                    end
                end
                -- Fallback: procura botões com "hatch"
                for _, gui in ipairs(LP.PlayerGui:GetDescendants()) do
                    if gui:IsA("TextButton") then
                        local txt = string.lower(gui.Text)
                        if string.find(txt, "hatch") or string.find(txt, "abrir") or 
                           string.find(txt, "chocar") or string.find(txt, "open") then
                            gui:Activate()
                        end
                    end
                end
            end)
        end
    end
end)

-- Auto Equip
task.spawn(function()
    while task.wait(1) do
        if Config.AutoEquip and Remotes.Equip then
            pcall(function()
                for _, obj in ipairs(Remotes.Equip:GetChildren()) do
                    if obj:IsA("StringValue") or obj:IsA("ObjectValue") then
                        Remotes.Equip:FireServer(obj.Value)
                    end
                end
            end)
        end
    end
end)

-- Auto Favorite
task.spawn(function()
    while task.wait(1) do
        if Config.AutoFavorite and Remotes.Favorite then
            pcall(function()
                Remotes.Favorite:FireServer()
            end)
        end
    end
end)

-- Auto Collect
task.spawn(function()
    while task.wait(0.3) do
        if Config.AutoCollect then
            pcall(function()
                -- Tenta via Remote
                if Remotes.Collect then
                    Remotes.Collect:FireServer()
                end
                -- ClickDetectors próximos
                local char = LP.Character
                if char then
                    local hrp = char:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        for _, obj in ipairs(workspace:GetDescendants()) do
                            if obj:IsA("ClickDetector") then
                                local parent = obj.Parent
                                if parent and parent:IsA("BasePart") then
                                    if (hrp.Position - parent.Position).Magnitude < 30 then
                                        fireclickdetector(obj)
                                    end
                                end
                            end
                        end
                    end
                end
            end)
        end
    end
end)

-- Auto Buy
task.spawn(function()
    while task.wait(1) do
        if Config.AutoBuy and Remotes.Buy then
            pcall(function()
                for _, gui in ipairs(LP.PlayerGui:GetDescendants()) do
                    if gui:IsA("TextButton") then
                        local txt = string.lower(gui.Text)
                        if string.find(txt, "buy") or string.find(txt, "comprar") then
                            gui:Activate()
                        end
                    end
                end
            end)
        end
    end
end)

-- Auto Sell
task.spawn(function()
    while task.wait(2) do
        if Config.AutoSell then
            pcall(function()
                if Remotes.Sell then
                    Remotes.Sell:FireServer(Config.SellFiltroRaridade)
                end
                -- Fallback: botões de venda
                for _, gui in ipairs(LP.PlayerGui:GetDescendants()) do
                    if gui:IsA("TextButton") then
                        local txt = string.lower(gui.Text)
                        if string.find(txt, "sell") or string.find(txt, "vender") then
                            gui:Activate()
                        end
                    end
                end
            end)
        end
    end
end)

-- Auto Steal (função específica do jogo)
task.spawn(function()
    while task.wait(0.2) do
        if Config.AutoSteal then
            pcall(function()
                local char = LP.Character
                if not char then return end
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if not hrp then return end
                
                -- Procura ovos próximos para roubar
                local alvo = nil
                local menorDist = Config.StealRange
                
                for _, obj in ipairs(workspace:GetDescendants()) do
                    if obj:IsA("BasePart") then
                        local n = string.lower(obj.Name)
                        if string.find(n, "egg") or string.find(n, "ovo") then
                            local dist = (hrp.Position - obj.Position).Magnitude
                            if dist < menorDist then
                                local raridade, valor = ObterDadosItem(obj)
                                if PassaFiltro(raridade, valor, Config.AutoStealFiltroRaridade, 0) then
                                    menorDist = dist
                                    alvo = obj
                                end
                            end
                        end
                    end
                end
                
                if alvo then
                    -- Tenta via remote
                    if Remotes.Steal then
                        Remotes.Steal:FireServer(alvo)
                    end
                    -- Fallback: teleporta até o ovo e interage
                    TeleportarPara(alvo.Position)
                    for _, c in ipairs(alvo:GetChildren()) do
                        if c:IsA("ClickDetector") then
                            fireclickdetector(c)
                        end
                    end
                end
            end)
        end
    end
end)

-- Auto Flee (fugir após roubar)
task.spawn(function()
    while task.wait(0.5) do
        if Config.AutoFlee then
            local char = LP.Character
            if char then
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if hrp and not hrp:FindFirstChild("EliteEggESP") then
                    -- Procura spawn mais distante
                    local spawns = {}
                    for _, obj in ipairs(workspace:GetDescendants()) do
                        if obj:IsA("SpawnLocation") then
                            table.insert(spawns, obj)
                        end
                    end
                    if #spawns > 0 then
                        local maisDistante = spawns[1]
                        local maxDist = 0
                        for _, sp in ipairs(spawns) do
                            local d = (hrp.Position - sp.Position).Magnitude
                            if d > maxDist then
                                maxDist = d
                                maisDistante = sp
                            end
                        end
                        if maxDist > 50 then
                            TeleportarPara(maisDistante.Position)
                        end
                    end
                end
            end
        end
    end
end)

-- Auto Reroll (boss)
task.spawn(function()
    while task.wait(2) do
        if Config.BossReroll and Remotes.Reroll then
            pcall(function()
                Remotes.Reroll:FireServer()
            end)
        end
    end
end)

-- Auto Recompensa
task.spawn(function()
    while task.wait(2) do
        if Config.AutoRecompensa then
            pcall(function()
                if Remotes.Reward then
                    Remotes.Reward:FireServer()
                end
                for _, gui in ipairs(LP.PlayerGui:GetDescendants()) do
                    if gui:IsA("TextButton") then
                        local txt = string.lower(gui.Text)
                        if string.find(txt, "claim") or string.find(txt, "recompensa") or 
                           string.find(txt, "coletar") then
                            gui:Activate()
                        end
                    end
                end
            end)
        end
    end
end)

--// ============ FPS/PING ============
local fpsContador = 0
local tempoAnterior = tick()

RunService.RenderStepped:Connect(function()
    fpsContador = fpsContador + 1
    local agora = tick()
    if agora - tempoAnterior >= 1 then
        if Config.MostrarFPS then
            FpsLabel.Text = "FPS: " .. fpsContador
            FpsLabel.TextColor3 = fpsContador >= 50 and Color3.fromRGB(100, 220, 120) 
                or fpsContador >= 30 and Color3.fromRGB(255, 200, 80) 
                or Color3.fromRGB(255, 80, 80)
        else
            FpsLabel.Text = ""
        end
        
        if Config.MostrarPing then
            local ping = 0
            pcall(function()
                ping = math.floor(game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValue())
            end)
            PingLabel.Text = "Ping: " .. ping .. " ms"
        else
            PingLabel.Text = ""
        end
        
        fpsContador = 0
        tempoAnterior = agora
    end
end)

--// ============ KEYBINDS ============
local guiAberta = true

FecharBtn.MouseButton1Click:Connect(function()
    guiAberta = false
    Main.Visible = false
end)

MinimizarBtn.MouseButton1Click:Connect(function()
    Main.Visible = false
    guiAberta = false
end)

UserInputService.InputBegan:Connect(function(input, processado)
    if processado then return end
    if input.KeyCode == Enum.KeyCode.RightShift then
        guiAberta = not guiAberta
        Main.Visible = guiAberta
        if guiAberta then
            Notificar("Elite Hub", "Interface aberta", 1)
        end
    end
end)

--// ============ INICIALIZAÇÃO ============
-- Abre primeira aba
Abas["ESP"].Botao:MouseButton1Click:Fire()

-- Carrega config salva
if isfile and isfile(CONFIG_FILE) then
    local ok, dados = pcall(function() return HttpService:JSONDecode(readfile(CONFIG_FILE)) end)
    if ok and dados then
        for k, v in pairs(dados) do Config[k] = v end
    end
end

Notificar("⚡ ELITE HUB", "Carregado com sucesso!\nKeybind: RightShift", 5)

print("═══════════════════════════════════════════")
print("  ⚡ ELITE HUB - Carregado!")
print("  Keybind: RightShift")
print("  Versão: 2.0.0")
print("═══════════════════════════════════════════")
