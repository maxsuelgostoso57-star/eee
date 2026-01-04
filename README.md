-- COMBAT HUB - FAST ATTACK & AIMBOT AVANÇADO
-- Sistema completo com Force Hit

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ============================================
-- CONFIGURAÇÕES GLOBAIS
-- ============================================
local Config = {
    -- Fast Attack
    FastAttackEnabled = false,
    AutoClick = false,
    ClickDelay = 0,
    AttackDistance = 100,
    
    -- Aimbot
    AimbotEnabled = false,
    FOVEnabled = true,
    FOVFixed = false, -- Pega só dentro do FOV
    FOV_SIZE = 200,
    MAX_DIST = 300,
    ForceHit = false, -- Força o hit modificando ataques
    ShowDebug = false, -- Mostra quando detecta projectiles
    
    -- ESP
    ESPEnabled = false,
    ESPPlayers = false,
    ESPDistance = false,
    ESPHealth = false
}

-- ============================================
-- GUI FLUENT
-- ============================================
local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

local Window = Fluent:CreateWindow({
    Title = "Combat Hub - Premium",
    SubTitle = "Advanced Combat System",
    TabWidth = 160,
    Theme = "Dark",
    Acrylic = false,
    Size = UDim2.fromOffset(500, 350),
    MinimizeKey = Enum.KeyCode.End
})

-- Criar Tabs
local Tabs = {
    Home = Window:AddTab({Title = "🏠 Home", Icon = "home"}),
    Combat = Window:AddTab({Title = "⚔️ Combat", Icon = "sword"}),
    Aimbot = Window:AddTab({Title = "🎯 Aimbot", Icon = "target"}),
    ESP = Window:AddTab({Title = "👁️ ESP", Icon = "eye"}),
    Settings = Window:AddTab({Title = "⚙️ Settings", Icon = "settings"})
}

-- ============================================
-- ABA HOME
-- ============================================
Tabs.Home:AddParagraph({
    Title = "👋 Bem-vindo ao Combat Hub!",
    Content = "Sistema premium de combat com Force Hit ❤️"
})

Tabs.Home:AddParagraph({
    Title = "👤 Jogador",
    Content = "Nome: " .. LocalPlayer.Name .. "\nDisplay: " .. LocalPlayer.DisplayName
})

Tabs.Home:AddParagraph({
    Title = "📋 Controles",
    Content = [[
🎯 AIMBOT:
• Aimbot Livre = Pega o mais próximo (qualquer direção)
• FOV Fixo = Só pega dentro do círculo
• Force Hit = Força seus ataques acertarem
• Debug Mode = F9 para ver console e testar

⚔️ COMBAT:
• Fast Attack = ataque automático
• Funciona junto com Aimbot e Force Hit

❓ NÃO FUNCIONA?
• Ative Debug Mode e aperte F9
• Use suas skills e veja se aparece no console
]]
})

-- ============================================
-- SISTEMA FAST ATTACK
-- ============================================
local FastAttack = {}

local function SafeWaitForChild(parent, childName)
    local success, result = pcall(function()
        return parent:WaitForChild(childName, 5)
    end)
    return success and result or nil
end

local Modules = SafeWaitForChild(ReplicatedStorage, "Modules")
local Net = Modules and SafeWaitForChild(Modules, "Net")
local RegisterAttack = Net and SafeWaitForChild(Net, "RE/RegisterAttack")
local RegisterHit = Net and SafeWaitForChild(Net, "RE/RegisterHit")

local function IsAlive(character)
    return character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0
end

local function ProcessEnemies(OthersEnemies, Folder)
    local BasePart = nil
    for _, Enemy in ipairs(Folder:GetChildren()) do
        local Head = Enemy:FindFirstChild("Head")
        if Head and IsAlive(Enemy) and LocalPlayer:DistanceFromCharacter(Head.Position) < Config.AttackDistance then
            if Enemy ~= LocalPlayer.Character then
                table.insert(OthersEnemies, { Enemy, Head })
                BasePart = Head
            end
        end
    end
    return BasePart
end

function FastAttack:Attack(BasePart, OthersEnemies)
    if not BasePart or #OthersEnemies == 0 or not RegisterAttack or not RegisterHit then return end
    pcall(function()
        RegisterAttack:FireServer(Config.ClickDelay or 0)
        RegisterHit:FireServer(BasePart, OthersEnemies)
    end)
end

function FastAttack:AttackNearest()
    if not Config.FastAttackEnabled then return end
    
    local OthersEnemies = {}
    local EnemiesFolder = workspace:FindFirstChild("Enemies")
    local CharactersFolder = workspace:FindFirstChild("Characters")
    local Part1 = nil
    local Part2 = nil
    
    if EnemiesFolder then Part1 = ProcessEnemies(OthersEnemies, EnemiesFolder) end
    if CharactersFolder then Part2 = ProcessEnemies(OthersEnemies, CharactersFolder) end
    
    local character = LocalPlayer.Character
    if not character then return end
    
    local equippedWeapon = character:FindFirstChildOfClass("Tool")
    if equippedWeapon and equippedWeapon:FindFirstChild("LeftClickRemote") then
        for _, enemyData in ipairs(OthersEnemies) do
            local enemy = enemyData[1]
            if enemy and enemy:FindFirstChild("HumanoidRootPart") and character:FindFirstChild("HumanoidRootPart") then
                local direction = (enemy.HumanoidRootPart.Position - character:GetPivot().Position).Unit
                pcall(function()
                    equippedWeapon.LeftClickRemote:FireServer(direction, 1)
                end)
            end
        end
    elseif #OthersEnemies > 0 then
        self:Attack(Part1 or Part2, OthersEnemies)
    end
end

function FastAttack:BladeHits()
    local Equipped = IsAlive(LocalPlayer.Character) and LocalPlayer.Character:FindFirstChildOfClass("Tool")
    if Equipped and Equipped.ToolTip ~= "Gun" then
        self:AttackNearest()
    end
end

-- Loop Fast Attack
task.spawn(function()
    while task.wait(Config.ClickDelay) do
        if Config.AutoClick and Config.FastAttackEnabled then
            FastAttack:BladeHits()
        end
    end
end)

-- ============================================
-- SISTEMA AIMBOT AVANÇADO
-- ============================================
local currentTarget = nil
local fovCircle = nil
local targetText = nil

-- Criar FOV Circle e Target Text
local function CreateFOV()
    if fovCircle then return end
    
    fovCircle = Drawing.new("Circle")
    fovCircle.Thickness = 2
    fovCircle.NumSides = 64
    fovCircle.Radius = Config.FOV_SIZE
    fovCircle.Filled = false
    fovCircle.Color = Color3.fromRGB(255, 255, 255)
    fovCircle.Transparency = 0.7
    fovCircle.Visible = Config.FOVEnabled
    
    targetText = Drawing.new("Text")
    targetText.Size = 20
    targetText.Center = true
    targetText.Outline = true
    targetText.Color = Color3.fromRGB(255, 255, 255)
    targetText.Visible = false
end

CreateFOV()

-- Pegar Root do player local
local function GetRoot()
    if LocalPlayer.Character then
        return LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    end
    return nil
end

-- Achar player mais próximo
local function GetTarget()
    local target = nil
    local dist = Config.MAX_DIST
    local Root = GetRoot()
    if not Root then return nil end
    
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local h = p.Character:FindFirstChild("Humanoid")
            local r = p.Character:FindFirstChild("HumanoidRootPart")
            if h and r and h.Health > 0 then
                local d = (r.Position - Root.Position).Magnitude
                
                -- MODO FOV FIXO (ON) - Só pega quem tá dentro do círculo FOV
                if Config.FOVFixed then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(r.Position)
                    if onScreen then
                        local mousePos = UserInputService:GetMouseLocation()
                        local screenDist = (Vector2.new(screenPos.X, screenPos.Y) - Vector2.new(mousePos.X, mousePos.Y)).Magnitude
                        
                        -- Verifica se está dentro do FOV E é o mais próximo
                        if screenDist <= Config.FOV_SIZE and d < dist then
                            dist = d
                            target = r
                        end
                    end
                else
                    -- MODO AIM LIVRE (OFF) - Pega o mais próximo ignorando FOV
                    if d < dist then
                        dist = d
                        target = r
                    end
                end
            end
        end
    end
    return target
end

-- Atualizar alvo constantemente
RunService.Heartbeat:Connect(function()
    if Config.AimbotEnabled then
        currentTarget = GetTarget()
    else
        currentTarget = nil
    end
    
    -- Atualizar FOV visual
    if fovCircle then
        fovCircle.Visible = Config.FOVEnabled and Config.AimbotEnabled
        fovCircle.Radius = Config.FOV_SIZE
        local mousePos = UserInputService:GetMouseLocation()
        fovCircle.Position = mousePos
        
        -- Mudar cor baseado no estado
        if currentTarget then
            -- Verde = tem alvo mas não forçando
            -- Vermelho = tem alvo E force hit ativo
            if Config.ForceHit then
                fovCircle.Color = Color3.fromRGB(255, 0, 0) -- Vermelho
                fovCircle.Thickness = 3
            else
                fovCircle.Color = Color3.fromRGB(0, 255, 0) -- Verde
                fovCircle.Thickness = 2
            end
            
            -- Mostrar nome do alvo
            if targetText and currentTarget.Parent then
                local player = Players:GetPlayerFromCharacter(currentTarget.Parent)
                if player then
                    targetText.Text = "🎯 " .. player.Name
                    targetText.Position = Vector2.new(mousePos.X, mousePos.Y - Config.FOV_SIZE - 30)
                    targetText.Visible = true
                    targetText.Color = Config.ForceHit and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(0, 255, 0)
                else
                    targetText.Visible = false
                end
            end
        else
            fovCircle.Color = Color3.fromRGB(255, 255, 255) -- Branco
            fovCircle.Thickness = 2
            if targetText then
                targetText.Visible = false
            end
        end
    end
end)

-- ============================================
-- FORCE HIT SYSTEM - HOOK REMOTES
-- ============================================
local old
old = hookmetamethod(game, "__namecall", function(self, ...)
    local args = {...}
    local method = getnamecallmethod()
    
    -- Intercepta FireServer e InvokeServer
    if not checkcaller() and (method == "FireServer" or method == "InvokeServer") then
        if currentTarget and Config.AimbotEnabled and Config.ForceHit then
            -- Debug
            if Config.ShowDebug then
                print("[FORCE HIT] Hook detectou:", self.Name, method)
            end
            
            -- Modifica todos os argumentos relevantes
            for i, v in pairs(args) do
                if typeof(v) == "Vector3" then
                    -- Substitui posições por posição do alvo
                    args[i] = currentTarget.Position
                elseif typeof(v) == "CFrame" then
                    -- Substitui CFrames pelo CFrame do alvo
                    args[i] = currentTarget.CFrame
                elseif typeof(v) == "Instance" and v ~= LocalPlayer then
                    -- Substitui instâncias pelo character do alvo
                    if v:IsA("Model") or v:IsA("BasePart") then
                        args[i] = currentTarget.Parent
                    end
                end
            end
        end
    end
    
    return old(self, unpack(args))
end)

-- FORÇA O HIT - Modifica as Parts dos seus ataques
local function ModifyAttackPart(part)
    if not part:IsA("BasePart") then return end
    if not Config.ForceHit or not Config.AimbotEnabled then return end
    if not currentTarget then return end
    
    -- Configurações do projectile
    part.CanCollide = false
    part.Massless = true
    
    -- Expande hitbox (invisível)
    local originalSize = part.Size
    part.Size = originalSize * 2.5
    
    -- Força direção pro alvo continuamente
    task.spawn(function()
        for i = 1, 150 do
            if not part.Parent or not currentTarget then break end
            
            local direction = (currentTarget.Position - part.Position).Unit
            
            -- Tenta modificar velocidade existente
            local velocity = part:FindFirstChildOfClass("BodyVelocity") or part:FindFirstChildOfClass("BodyForce")
            
            if velocity then
                if velocity:IsA("BodyVelocity") then
                    velocity.Velocity = direction * 100
                end
            else
                -- Cria nova velocidade
                local bv = Instance.new("BodyVelocity")
                bv.Velocity = direction * 100
                bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                bv.Parent = part
            end
            
            -- Teleporta se muito longe
            local distToTarget = (part.Position - currentTarget.Position).Magnitude
            if distToTarget > 40 then
                part.CFrame = part.CFrame:Lerp(currentTarget.CFrame, 0.3)
            end
            
            task.wait(0.02)
        end
    end)
    
    -- Detecta colisão com alvo
    local conn
    conn = part.Touched:Connect(function(hit)
        if currentTarget and hit:IsDescendantOf(currentTarget.Parent) then
            -- Confirma hit teleportando
            part.CFrame = currentTarget.CFrame
            task.wait(0.03)
        end
    end)
    
    task.delay(5, function()
        if conn then conn:Disconnect() end
    end)
end

-- Detecta quando você cria um ataque/skill
workspace.DescendantAdded:Connect(function(obj)
    if not Config.ForceHit or not Config.AimbotEnabled then return end
    if not obj:IsA("BasePart") then return end
    if not currentTarget then return end
    
    local Root = GetRoot()
    if not Root then return end
    
    -- Detecta se é um ataque seu
    local isYourAttack = false
    local objName = obj.Name:lower()
    
    -- Método 1: Nomes comuns de skills/ataques
    local attackKeywords = {
        "effect", "skill", "attack", "projectile", "blast", 
        "ball", "slash", "strike", "punch", "kick",
        "beam", "ray", "bullet", "missile", "sword"
    }
    
    for _, keyword in ipairs(attackKeywords) do
        if string.find(objName, keyword) then
            if (obj.Position - Root.Position).Magnitude < 50 then
                isYourAttack = true
                break
            end
        end
    end
    
    -- Método 2: Verifica propriedade Owner
    if not isYourAttack then
        pcall(function()
            if obj:FindFirstChild("Owner") and obj.Owner.Value == LocalPlayer then
                isYourAttack = true
            end
        end)
    end
    
    -- Método 3: Material especial (Neon/ForceField)
    if not isYourAttack then
        if obj.Material == Enum.Material.Neon or obj.Material == Enum.Material.ForceField then
            if (obj.Position - Root.Position).Magnitude < 50 then
                isYourAttack = true
            end
        end
    end
    
    -- Método 4: Verifica se está perto de você e tem BodyVelocity
    if not isYourAttack then
        if obj:FindFirstChildOfClass("BodyVelocity") or obj:FindFirstChildOfClass("BodyForce") then
            if (obj.Position - Root.Position).Magnitude < 40 then
                isYourAttack = true
            end
        end
    end
    
    -- Aplica force hit se for seu ataque
    if isYourAttack and currentTarget then
        if Config.ShowDebug then
            print("[FORCE HIT] Detectou projectile:", obj.Name)
        end
        task.wait(0.05) -- Pequeno delay para garantir que o projectile inicializou
        ModifyAttackPart(obj)
    end
end)

-- ============================================
-- SISTEMA ESP
-- ============================================
local ESPObjects = {}

local function CreateESP(player)
    if not player.Character then return end
    
    local humanoidRootPart = player.Character:FindFirstChild("HumanoidRootPart")
    if not humanoidRootPart then return end
    
    -- Remover ESP antigo se existir
    if ESPObjects[player] then
        for _, obj in pairs(ESPObjects[player]) do
            if obj then obj:Destroy() end
        end
    end
    
    ESPObjects[player] = {}
    
    -- Box ESP
    local box = Drawing.new("Square")
    box.Visible = false
    box.Color = Color3.fromRGB(255, 0, 0)
    box.Thickness = 2
    box.Transparency = 1
    box.Filled = false
    ESPObjects[player].Box = box
    
    -- Name Text
    local nameText = Drawing.new("Text")
    nameText.Visible = false
    nameText.Color = Color3.fromRGB(255, 255, 255)
    nameText.Size = 18
    nameText.Center = true
    nameText.Outline = true
    nameText.Text = player.Name
    ESPObjects[player].Name = nameText
    
    -- Distance Text
    local distanceText = Drawing.new("Text")
    distanceText.Visible = false
    distanceText.Color = Color3.fromRGB(255, 255, 0)
    distanceText.Size = 16
    distanceText.Center = true
    distanceText.Outline = true
    ESPObjects[player].Distance = distanceText
    
    -- Health Text
    local healthText = Drawing.new("Text")
    healthText.Visible = false
    healthText.Color = Color3.fromRGB(0, 255, 0)
    healthText.Size = 16
    healthText.Center = true
    healthText.Outline = true
    ESPObjects[player].Health = healthText
end

local function UpdateESP()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            local humanoid = player.Character:FindFirstChild("Humanoid")
            local rootPart = player.Character:FindFirstChild("HumanoidRootPart")
            
            if humanoid and rootPart and humanoid.Health > 0 then
                if not ESPObjects[player] then
                    CreateESP(player)
                end
                
                local screenPos, onScreen = Camera:WorldToViewportPoint(rootPart.Position)
                
                if onScreen and ESPObjects[player] then
                    local Root = GetRoot()
                    if not Root then return end
                    
                    -- Calcular distância
                    local distance = (rootPart.Position - Root.Position).Magnitude
                    
                    -- Atualizar Box
                    if ESPObjects[player].Box then
                        ESPObjects[player].Box.Visible = Config.ESPEnabled and Config.ESPPlayers
                        if ESPObjects[player].Box.Visible then
                            local headPos = Camera:WorldToViewportPoint(player.Character.Head.Position + Vector3.new(0, 0.5, 0))
                            local legPos = Camera:WorldToViewportPoint(rootPart.Position - Vector3.new(0, 3, 0))
                            
                            ESPObjects[player].Box.Size = Vector2.new(2000 / screenPos.Z, headPos.Y - legPos.Y)
                            ESPObjects[player].Box.Position = Vector2.new(screenPos.X - ESPObjects[player].Box.Size.X / 2, legPos.Y)
                        end
                    end
                    
                    -- Atualizar Nome
                    if ESPObjects[player].Name then
                        ESPObjects[player].Name.Visible = Config.ESPEnabled and Config.ESPPlayers
                        if ESPObjects[player].Name.Visible then
                            ESPObjects[player].Name.Position = Vector2.new(screenPos.X, screenPos.Y - 40)
                        end
                    end
                    
                    -- Atualizar Distância
                    if ESPObjects[player].Distance then
                        ESPObjects[player].Distance.Visible = Config.ESPEnabled and Config.ESPDistance
                        if ESPObjects[player].Distance.Visible then
                            ESPObjects[player].Distance.Text = math.floor(distance) .. "m"
                            ESPObjects[player].Distance.Position = Vector2.new(screenPos.X, screenPos.Y - 20)
                        end
                    end
                    
                    -- Atualizar Health
                    if ESPObjects[player].Health then
                        ESPObjects[player].Health.Visible = Config.ESPEnabled and Config.ESPHealth
                        if ESPObjects[player].Health.Visible then
                            ESPObjects[player].Health.Text = math.floor(humanoid.Health) .. " HP"
                            ESPObjects[player].Health.Position = Vector2.new(screenPos.X, screenPos.Y)
                        end
                    end
                else
                    -- Ocultar se fora da tela
                    if ESPObjects[player] then
                        for _, obj in pairs(ESPObjects[player]) do
                            if obj then obj.Visible = false end
                        end
                    end
                end
            end
        end
    end
end

-- Loop ESP
RunService.RenderStepped:Connect(function()
    if Config.ESPEnabled then
        UpdateESP()
    else
        for _, espData in pairs(ESPObjects) do
            for _, obj in pairs(espData) do
                if obj then obj.Visible = false end
            end
        end
    end
end)

-- Remover ESP quando player sair
Players.PlayerRemoving:Connect(function(player)
    if ESPObjects[player] then
        for _, obj in pairs(ESPObjects[player]) do
            if obj then obj:Destroy() end
        end
        ESPObjects[player] = nil
    end
end)

-- ============================================
-- ABA COMBAT
-- ============================================
Tabs.Combat:AddSection("⚔️ Fast Attack System")

Tabs.Combat:AddToggle("FastAttack", {
    Title = "🗡️ Fast Attack",
    Description = "Ativa ataque rápido automático",
    Default = false,
    Callback = function(Value)
        Config.FastAttackEnabled = Value
        Config.AutoClick = Value
        Fluent:Notify({
            Title = "Fast Attack",
            Content = Value and "Ativado ✓" or "Desativado ✗",
            Duration = 3
        })
    end
})

Tabs.Combat:AddSlider("AttackDistance", {
    Title = "📏 Distância de Ataque",
    Description = "Alcance máximo para atacar",
    Default = 100,
    Min = 50,
    Max = 300,
    Rounding = 0,
    Callback = function(Value)
        Config.AttackDistance = Value
    end
})

Tabs.Combat:AddSlider("ClickDelay", {
    Title = "⏱️ Delay de Ataque",
    Description = "Intervalo entre ataques (ms)",
    Default = 0,
    Min = 0,
    Max = 100,
    Rounding = 0,
    Callback = function(Value)
        Config.ClickDelay = Value / 1000
    end
})

-- ============================================
-- ABA AIMBOT
-- ============================================
Tabs.Aimbot:AddSection("🎯 Sistema de Aimbot")

Tabs.Aimbot:AddToggle("AimbotEnabled", {
    Title = "🎯 Aimbot",
    Description = "Ativa o sistema de aimbot",
    Default = false,
    Callback = function(Value)
        Config.AimbotEnabled = Value
        Fluent:Notify({
            Title = "Aimbot",
            Content = Value and "Ativado ✓" or "Desativado ✗",
            Duration = 3
        })
    end
})

Tabs.Aimbot:AddToggle("FOVEnabled", {
    Title = "👁️ Mostrar FOV",
    Description = "Exibe círculo de FOV na tela",
    Default = true,
    Callback = function(Value)
        Config.FOVEnabled = Value
        if fovCircle then
            fovCircle.Visible = Value and Config.AimbotEnabled
        end
    end
})

Tabs.Aimbot:AddToggle("FOVFixed", {
    Title = "🔒 FOV Fixo",
    Description = "ON = Só pega dentro do FOV | OFF = Pega qualquer um próximo",
    Default = false,
    Callback = function(Value)
        Config.FOVFixed = Value
        Fluent:Notify({
            Title = "FOV Fixo",
            Content = Value and "FIXO - Só dentro do círculo ✓" or "LIVRE - Pega o mais próximo ✗",
            Duration = 4
        })
    end
})

Tabs.Aimbot:AddToggle("ForceHit", {
    Title = "💥 Force Hit",
    Description = "FORÇA seus ataques/skills acertarem no alvo",
    Default = false,
    Callback = function(Value)
        Config.ForceHit = Value
        Fluent:Notify({
            Title = "Force Hit",
            Content = Value and "✓ Ativado - Seus ataques vão pro alvo!" or "✗ Desativado",
            Duration = 3
        })
    end
})

Tabs.Aimbot:AddSlider("FOVSize", {
    Title = "📐 Tamanho FOV",
    Description = "Tamanho do círculo de visão",
    Default = 200,
    Min = 50,
    Max = 500,
    Rounding = 0,
    Callback = function(Value)
        Config.FOV_SIZE = Value
    end
})

Tabs.Aimbot:AddSlider("MaxDistance", {
    Title = "📏 Distância Máxima",
    Description = "Alcance máximo do aimbot",
    Default = 300,
    Min = 100,
    Max = 1000,
    Rounding = 0,
    Callback = function(Value)
        Config.MAX_DIST = Value
    end
})

Tabs.Aimbot:AddToggle("ShowDebug", {
    Title = "🐛 Debug Mode",
    Description = "Mostra no console quando detecta projectiles (F9 para ver)",
    Default = false,
    Callback = function(Value)
        Config.ShowDebug = Value
        Fluent:Notify({
            Title = "Debug Mode",
            Content = Value and "✓ ON - Aperte F9 para ver console" or "✗ OFF",
            Duration = 3
        })
    end
})

Tabs.Aimbot:AddParagraph({
    Title = "ℹ️ Modos de Aimbot",
    Content = [[
🎯 FOV FIXO (ON):
• Só trava em quem está DENTRO do círculo
• Mais preciso e seguro

🌐 AIM LIVRE (OFF):
• Pega o player mais PRÓXIMO
• Ignora o círculo FOV
• Pega em qualquer direção

💥 FORCE HIT:
• Seus ataques sempre acertam
• Modifica projectiles automaticamente

🐛 DEBUG MODE:
• Ative para ver no console (F9)
• Mostra quando detecta seus ataques
• Útil para testar se está funcionando
]]
})

-- ============================================
-- ABA ESP
-- ============================================
Tabs.ESP:AddSection("👁️ ESP System")

Tabs.ESP:AddToggle("ESPEnabled", {
    Title = "👁️ Ativar ESP",
    Description = "Liga/Desliga o sistema ESP",
    Default = false,
    Callback = function(Value)
        Config.ESPEnabled = Value
        Fluent:Notify({
            Title = "ESP",
            Content = Value and "Ativado ✓" or "Desativado ✗",
            Duration = 3
        })
    end
})

Tabs.ESP:AddToggle("ESPPlayers", {
    Title = "🧑 ESP Jogadores",
    Description = "Mostra box e nome dos jogadores",
    Default = false,
    Callback = function(Value)
        Config.ESPPlayers = Value
    end
})

Tabs.ESP:AddToggle("ESPDistance", {
    Title = "📏 Mostrar Distância",
    Description = "Exibe distância dos jogadores",
    Default = false,
    Callback = function(Value)
        Config.ESPDistance = Value
    end
})

Tabs.ESP:AddToggle("ESPHealth", {
    Title = "❤️ Mostrar HP",
    Description = "Exibe vida dos jogadores",
    Default = false,
    Callback = function(Value)
        Config.ESPHealth = Value
    end
})

-- ============================================
-- ABA SETTINGS
-- ============================================
Tabs.Settings:AddSection("⚙️ Configurações")

Tabs.Settings:AddButton({
    Title = "🔄 Resetar Tudo",
    Description = "Restaura configurações padrão",
    Callback = function()
        Config.FOV_SIZE = 200
        Config.MAX_DIST = 300
        Config.AttackDistance = 100
        Config.ClickDelay = 0
        Config.FOVFixed = false
        Config.ForceHit = false
        
        Fluent:Notify({
            Title = "Settings",
            Content = "Configurações resetadas!",
            Duration = 3
        })
    end
})

-- ============================================
-- BOTÃO TOGGLE GUI
-- ============================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "CombatHubGUI"
screenGui.Parent = game.CoreGui

local toggleButton = Instance.new("ImageButton")
toggleButton.Size = UDim2.new(0, 50, 0, 50)
toggleButton.Position = UDim2.new(0.15, 0, 0.15, 0)
toggleButton.Image = "rbxassetid://136724157425040"
toggleButton.BackgroundTransparency = 1
toggleButton.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0.25, 0)
corner.Parent = toggleButton

local isFluentVisible = true
local dragging, dragInput, dragStart, startPos

local function update(input)
    local delta = input.Position - dragStart
    toggleButton.Position = UDim2.new(
        startPos.X.Scale,
        startPos.X.Offset + delta.X,
        startPos.Y.Scale,
        startPos.Y.Offset + delta.Y
    )
end

toggleButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch 
    or input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = toggleButton.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

toggleButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Touch 
    or input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        update(input)
    end
end)

toggleButton.MouseButton1Click:Connect(function()
    isFluentVisible = not isFluentVisible
    Window:Minimize(not isFluentVisible)
end)

-- ============================================
-- NOTIFICAÇÃO INICIAL
-- ============================================
Fluent:Notify({
    Title = "Combat Hub",
    Content = "Carregado! Bem-vindo " .. LocalPlayer.Name,
    Duration = 5
})

print("╔════════════════════════════════╗")
print("║   COMBAT HUB - LOADED         ║")
print("╠════════════════════════════════╣")
print("║  ✓ Fast Attack System         ║")
print("║  ✓ Advanced Aimbot            ║")
print("║  ✓ Force Hit System           ║")
print("║  ✓ ESP System                 ║")
print("╚════════════════════════════════╝")
