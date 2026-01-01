-- Servidor: TWWMiningSystem.server.lua
-- Sistema de Mineração para The Wild West

local ServerScriptService = game:GetService("ServerScriptService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local MarketplaceService = game:GetService("MarketplaceService")
local CollectionService = game:GetService("CollectionService")

-- Criar RemoteEvents
local RemoteEvents = Instance.new("Folder", ReplicatedStorage)
RemoteEvents.Name = "TWWRemoteEvents"

local CreateWaypointEvent = Instance.new("RemoteEvent", RemoteEvents)
CreateWaypointEvent.Name = "CreateWaypoint"

local StartMiningEvent = Instance.new("RemoteEvent", RemoteEvents)
StartMiningEvent.Name = "StartMining"

local DeletePathEvent = Instance.new("RemoteEvent", RemoteEvents)
DeletePathEvent.Name = "DeletePath"

local BuyMuleEvent = Instance.new("RemoteEvent", RemoteEvents)
BuyMuleEvent.Name = "BuyMule"

-- Configurações específicas para The Wild West
local CONFIG = {
    MaxWaypoints = 15,
    MiningTime = 5, -- segundos (mais realista para TWW)
    NPCWalkSpeed = 12, -- Velocidade realista para mulas
    MinHeight = 1, -- Altura das bolinhas
    MaxDistanceFromPlayer = 100, -- Distância máxima para criar waypoints
    MulePrice = 50, -- Preço da mula em dólares
}

-- Minérios baseados em The Wild West
local ORES = {
    ["GoldNugget"] = {
        Name = "Pepita de Ouro",
        Value = 10, -- Valor base
        Color = Color3.fromRGB(255, 215, 0),
        Rarity = 0.05,
        Weight = 0.5, -- Peso em libras
        ModelId = "rbxassetid://" -- ID do modelo da pepita
    },
    ["SilverOre"] = {
        Name = "Minério de Prata",
        Value = 25,
        Color = Color3.fromRGB(192, 192, 192),
        Rarity = 0.15,
        Weight = 2.0
    },
    ["CopperOre"] = {
        Name = "Minério de Cobre",
        Value = 15,
        Color = Color3.fromRGB(184, 115, 51),
        Rarity = 0.30,
        Weight = 3.0
    },
    ["Coal"] = {
        Name = "Carvão",
        Value = 5,
        Color = Color3.fromRGB(30, 30, 30),
        Rarity = 0.50,
        Weight = 1.0
    }
}

-- Sistema de inventário simulando TWW
local PlayerInventory = {}

-- Função para criar dados do jogador no estilo TWW
local function setupPlayerData(player)
    -- Se já existir, não criar novamente
    if PlayerInventory[player.UserId] then return end
    
    PlayerInventory[player.UserId] = {
        Money = 100, -- Dinheiro inicial
        Mules = 0,
        MaxMules = 3,
        Tools = {"Pickaxe"},
        CurrentLoad = 0,
        MaxLoad = 50, -- Peso máximo em libras
        MiningLevel = 1,
        Experience = 0
    }
    
    -- Criar valores na GUI
    local leaderstats = Instance.new("Folder")
    leaderstats.Name = "leaderstats"
    leaderstats.Parent = player
    
    local money = Instance.new("IntValue")
    money.Name = "Money"
    money.Value = PlayerInventory[player.UserId].Money
    money.Parent = leaderstats
    
    local mules = Instance.new("IntValue")
    mules.Name = "Mules"
    mules.Value = PlayerInventory[player.UserId].Mules
    mules.Parent = leaderstats
    
    local load = Instance.new("IntValue")
    load.Name = "CurrentLoad"
    load.Value = PlayerInventory[player.UserId].CurrentLoad
    load.Parent = leaderstats
end

-- Função para criar waypoint no estilo western
local function createWaypoint(position, player)
    if not player.Character then return nil, "Personagem não encontrado" end
    
    -- Verificar distância do jogador
    local distance = (player.Character.HumanoidRootPart.Position - position).Magnitude
    if distance > CONFIG.MaxDistanceFromPlayer then
        return nil, "Muito longe! Aproxime-se mais."
    end
    
    if #waypoints >= CONFIG.MaxWaypoints then
        return nil, "Limite de " .. CONFIG.MaxWaypoints .. " marcos atingido!"
    end
    
    -- Criar waypoint no estilo western (marcação no chão)
    local waypoint = Instance.new("Part")
    waypoint.Name = "TrailMarker_" .. player.Name .. "_" .. #waypoints + 1
    waypoint.Size = Vector3.new(1.5, 0.2, 1.5)
    waypoint.Shape = Enum.PartType.Cylinder
    waypoint.Material = Enum.Material.WoodPlanks
    waypoint.Color = Color3.fromRGB(139, 69, 19) -- Marrom madeira
    waypoint.Position = position + Vector3.new(0, 0.1, 0)
    waypoint.Anchored = true
    waypoint.CanCollide = false
    waypoint.Transparency = 0
    waypoint.Rotation = Vector3.new(0, 0, 90)
    waypoint.Parent = Workspace
    
    -- Adicionar decalque de pegada ou símbolo
    local decal = Instance.new("Decal")
    decal.Name = "MarkerSymbol"
    decal.Texture = "rbxassetid://2781140922" -- Textura de pegada
    decal.Face = Enum.NormalId.Top
    decal.Parent = waypoint
    
    -- Sinalizador de bandeira
    local flagPole = Instance.new("Part")
    flagPole.Name = "FlagPole"
    flagPole.Size = Vector3.new(0.2, 3, 0.2)
    flagPole.Position = waypoint.Position + Vector3.new(0, 1.5, 0)
    flagPole.Color = Color3.fromRGB(100, 100, 100)
    flagPole.Material = Enum.Material.Metal
    flagPole.Anchored = true
    flagPole.CanCollide = false
    flagPole.Parent = waypoint
    
    local flag = Instance.new("Part")
    flag.Name = "Flag"
    flag.Size = Vector3.new(1, 0.5, 0.1)
    flag.Position = flagPole.Position + Vector3.new(0.5, 0, 0)
    flag.Color = Color3.fromRGB(200, 0, 0)
    flag.Material = Enum.Material.Fabric
    flag.Anchored = true
    flag.CanCollide = false
    flag.Parent = waypoint
    
    -- Attachment para beams
    local attachment = Instance.new("Attachment")
    attachment.Position = Vector3.new(0, 1, 0)
    attachment.Parent = waypoint
    
    table.insert(waypoints, {
        Part = waypoint,
        Attachment = attachment,
        Position = waypoint.Position,
        Owner = player
    })
    
    -- Criar trilha de fumaça/poeira conectando os pontos
    if #waypoints > 1 then
        local previousWaypoint = waypoints[#waypoints - 1]
        
        local trail = Instance.new("Beam")
        trail.Name = "Trail_" .. #waypoints
        trail.Attachment0 = previousWaypoint.Attachment
        trail.Attachment1 = attachment
        trail.Color = ColorSequence.new(Color3.fromRGB(160, 120, 80)) -- Cor de poeira
        trail.Width0 = 0.3
        trail.Width1 = 0.3
        trail.Texture = "rbxassetid://2809280113" -- Textura de poeira
        trail.TextureSpeed = 2
        trail.TextureLength = 2
        trail.FaceCamera = true
        trail.Transparency = NumberSequence.new(0.7)
        trail.Parent = Workspace
        
        table.insert(pathParts, trail)
    end
    
    -- Som de colocação de marker
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://140691535" -- Som de madeira batendo
    sound.Volume = 0.5
    sound.Parent = waypoint
    sound:Play()
    game:GetService("Debris"):AddItem(sound, 3)
    
    return waypoint
end

-- Função para criar mula de carga (no estilo TWW)
local function createMiningMule(player)
    local playerData = PlayerInventory[player.UserId]
    
    -- Verificar se pode ter mais mulas
    if playerData.Mules >= playerData.MaxMules then
        return nil, "Limite de mulas atingido!"
    end
    
    -- Criar modelo da mula
    local muleModel = Instance.new("Model")
    muleModel.Name = "MiningMule_" .. player.Name
    
    -- Corpo da mula
    local body = Instance.new("Part")
    body.Name = "Body"
    body.Size = Vector3.new(3, 2, 5)
    body.Color = Color3.fromRGB(101, 67, 33) -- Marrom da mula
    body.Material = Enum.Material.Fabric
    body.Anchored = false
    body.CanCollide = true
    body.Parent = muleModel
    
    -- Cabeça
    local head = Instance.new("Part")
    head.Name = "Head"
    head.Size = Vector3.new(1.5, 1.5, 2)
    head.Color = Color3.fromRGB(80, 50, 20)
    head.Position = body.Position + Vector3.new(0, 0, -3)
    head.Anchored = false
    head.CanCollide = true
    head.Parent = muleModel
    
    -- Orelhas
    local ear1 = Instance.new("Part")
    ear1.Name = "Ear1"
    ear1.Size = Vector3.new(0.2, 0.8, 0.2)
    ear1.Color = Color3.fromRGB(80, 50, 20)
    ear1.Parent = muleModel
    
    -- Carregamento (caixas)
    local cargo = Instance.new("Part")
    cargo.Name = "Cargo"
    cargo.Size = Vector3.new(2.5, 1.5, 3)
    cargo.Color = Color3.fromRGB(150, 100, 50)
    cargo.Material = Enum.Material.WoodPlanks
    cargo.Position = body.Position + Vector3.new(0, 1.5, 0)
    cargo.Anchored = false
    cargo.CanCollide = true
    cargo.Parent = muleModel
    
    -- Humanoid para movimento
    local humanoid = Instance.new("Humanoid")
    humanoid.WalkSpeed = CONFIG.NPCWalkSpeed
    humanoid.JumpPower = 0
    humanoid.Parent = muleModel
    
    -- Configurar welds
    local welds = {}
    
    for _, part in pairs(muleModel:GetChildren()) do
        if part:IsA("BasePart") and part.Name ~= "Body" then
            local weld = Instance.new("Weld")
            weld.Part0 = body
            weld.Part1 = part
            weld.C0 = CFrame.new()
            weld.Parent = body
            table.insert(welds, weld)
        end
    end
    
    -- Ajustar posições dos welds
    if welds[1] then welds[1].C0 = CFrame.new(0, 0.5, -3) end -- Cabeça
    if welds[2] then welds[2].C0 = CFrame.new(0.3, 0.8, -3.5) end -- Orelha1
    if welds[3] then welds[3].C0 = CFrame.new(0, 1.5, 0) end -- Carga
    
    muleModel.PrimaryPart = body
    muleModel.Parent = Workspace
    
    -- Tag para identificação
    CollectionService:AddTag(muleModel, "MiningMule")
    CollectionService:AddTag(muleModel, player.UserId .. "_Mule")
    
    playerData.Mules = playerData.Mules + 1
    
    -- Atualizar GUI
    local mulesStat = player.leaderstats:FindFirstChild("Mules")
    if mulesStat then
        mulesStat.Value = playerData.Mules
    end
    
    return muleModel, humanoid
end

-- Função de mineração com sistema de stamina/peso
local function mineOre(player, muleModel)
    local playerData = PlayerInventory[player.UserId]
    
    -- Verificar se tem picareta
    if not table.find(playerData.Tools, "Pickaxe") then
        return nil, "Você precisa de uma picareta!"
    end
    
    -- Verificar carga máxima
    if playerData.CurrentLoad >= playerData.MaxLoad then
        return nil, "Carga máxima atingida! Venda seus minérios."
    end
    
    -- Escolher minério baseado na raridade e nível
    local random = math.random()
    local selectedOre = nil
    
    for oreName, oreData in pairs(ORES) do
        local adjustedRarity = oreData.Rarity * playerData.MiningLevel
        if random <= adjustedRarity then
            selectedOre = oreName
            break
        end
        random = random - adjustedRarity
    end
    
    if not selectedOre then selectedOre = "Coal" end -- Padrão
    
    local oreData = ORES[selectedOre]
    
    -- Verificar se cabe na carga
    if playerData.CurrentLoad + oreData.Weight > playerData.MaxLoad then
        return nil, "Peso muito alto! Carga atual: " .. playerData.CurrentLoad .. "/" .. playerData.MaxLoad
    end
    
    -- Adicionar ao inventário
    playerData.CurrentLoad = playerData.CurrentLoad + oreData.Weight
    
    -- Ganhar experiência
    local expGained = math.floor(oreData.Value / 2)
    playerData.Experience = playerData.Experience + expGained
    
    -- Verificar nível up
    if playerData.Experience >= (playerData.MiningLevel * 100) then
        playerData.MiningLevel = playerData.MiningLevel + 1
        playerData.Experience = 0
        
        -- Feedback de nível up
        local levelBillboard = Instance.new("BillboardGui")
        levelBillboard.Size = UDim2.new(0, 300, 0, 50)
        levelBillboard.StudsOffset = Vector3.new(0, 4, 0)
        levelBillboard.Adornee = muleModel.PrimaryPart
        levelBillboard.Parent = muleModel.PrimaryPart
        
        local text = Instance.new("TextLabel")
        text.Size = UDim2.new(1, 0, 1, 0)
        text.Text = "🎉 Nível " .. playerData.MiningLevel .. " de Mineração!"
        text.TextColor3 = Color3.fromRGB(255, 215, 0)
        text.TextScaled = true
        text.BackgroundTransparency = 1
        text.Parent = levelBillboard
        
        game:GetService("Debris"):AddItem(levelBillboard, 3)
    end
    
    -- Atualizar GUI
    local loadStat = player.leaderstats:FindFirstChild("CurrentLoad")
    if loadStat then
        loadStat.Value = math.floor(playerData.CurrentLoad)
    end
    
    return oreData
end

-- Função para vender minérios
local function sellOres(player)
    local playerData = PlayerInventory[player.UserId]
    
    if playerData.CurrentLoad == 0 then
        return 0, "Nada para vender!"
    end
    
    -- Calcular valor total (simplificado)
    local totalValue = playerData.CurrentLoad * 15 -- Valor base por peso
    
    -- Bônus por nível
    totalValue = totalValue * (1 + (playerData.MiningLevel * 0.1))
    
    -- Arredondar
    totalValue = math.floor(totalValue)
    
    -- Adicionar dinheiro
    playerData.Money = playerData.Money + totalValue
    playerData.CurrentLoad = 0
    
    -- Atualizar GUI
    local moneyStat = player.leaderstats:FindFirstChild("Money")
    local loadStat = player.leaderstats:FindFirstChild("CurrentLoad")
    
    if moneyStat then moneyStat.Value = playerData.Money end
    if loadStat then loadStat.Value = playerData.CurrentLoad end
    
    return totalValue
end

-- Sistema principal de mineração
local waypoints = {}
local activeMules = {}
local isMiningActive = false
local pathParts = {}

local function startMiningRoute(player)
    if #waypoints == 0 or isMiningActive then 
        return false, "Nenhuma rota definida ou mineração já em andamento!"
    end
    
    local playerData = PlayerInventory[player.UserId]
    
    -- Verificar se tem mula
    if playerData.Mules == 0 then
        return false, "Você precisa de uma mula! Compre na loja."
    end
    
    -- Criar mula
    local muleModel, muleHumanoid = createMiningMule(player)
    if not muleModel then
        return false, "Não foi possível criar a mula!"
    end
    
    isMiningActive = true
    
    -- Posicionar mula no primeiro waypoint
    if waypoints[1] then
        muleModel:SetPrimaryPartCFrame(CFrame.new(waypoints[1].Position + Vector3.new(0, 2, 0)))
    end
    
    -- Seguir rota
    for i, waypointData in ipairs(waypoints) do
        local waypoint = waypointData.Part
        
        -- Mover mula
        muleHumanoid:MoveTo(waypoint.Position)
        
        -- Esperar chegada
        local arrived = false
        local connection
        
        connection = muleHumanoid.MoveToFinished:Connect(function(reached)
            if reached then
                arrived = true
                connection:Disconnect()
            end
        end)
        
        local startTime = tick()
        while not arrived and tick() - startTime < 15 do
            wait(0.1)
        end
        
        -- Se for o último waypoint, minerar
        if i == #waypoints then
            -- Animar mineração
            local miningCFrame = muleModel.PrimaryPart.CFrame
            local tweenInfo = TweenInfo.new(0.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
            local tween = TweenService:Create(muleModel.PrimaryPart, tweenInfo, {
                CFrame = miningCFrame * CFrame.new(0, 0.2, 0)
            })
            tween:Play()
            
            -- Tempo de mineração
            for time = 1, CONFIG.MiningTime do
                -- Feedback visual de progresso
                local progress = Instance.new("BillboardGui")
                progress.Size = UDim2.new(0, 100, 0, 20)
                progress.StudsOffset = Vector3.new(0, 3, 0)
                progress.Adornee = muleModel.PrimaryPart
                progress.Parent = muleModel.PrimaryPart
                
                local bar = Instance.new("Frame")
                bar.Size = UDim2.new(time/CONFIG.MiningTime, 0, 1, 0)
                bar.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
                bar.BorderSizePixel = 0
                bar.Parent = progress
                
                game:GetService("Debris"):AddItem(progress, 1)
                wait(1)
            end
            
            tween:Cancel()
            
            -- Mineração
            local oreData, errorMsg = mineOre(player, muleModel)
            
            if oreData then
                -- Feedback visual
                local oreBillboard = Instance.new("BillboardGui")
                oreBillboard.Size = UDim2.new(0, 200, 0, 50)
                oreBillboard.StudsOffset = Vector3.new(0, 2, 0)
                oreBillboard.Adornee = muleModel.PrimaryPart
                oreBillboard.Parent = muleModel.PrimaryPart
                
                local text = Instance.new("TextLabel")
                text.Size = UDim2.new(1, 0, 1, 0)
                text.Text = oreData.Name .. " +" .. oreData.Value .. "$"
                text.TextColor3 = oreData.Color
                text.TextScaled = true
                text.BackgroundTransparency = 1
                text.Parent = oreBillboard
                
                -- Som de mineração
                local sound = Instance.new("Sound")
                sound.SoundId = "rbxassetid://9081602397" -- Som de picareta
                sound.Volume = 0.7
                sound.Parent = muleModel.PrimaryPart
                sound:Play()
                
                game:GetService("Debris"):AddItem(oreBillboard, 2)
                game:GetService("Debris"):AddItem(sound, 3)
            end
            
            -- Venda automática se carga cheia
            if playerData.CurrentLoad >= playerData.MaxLoad * 0.8 then
                local saleValue = sellOres(player)
                
                if saleValue > 0 then
                    -- Feedback de venda
                    local saleBillboard = Instance.new("BillboardGui")
                    saleBillboard.Size = UDim2.new(0, 250, 0, 60)
                    saleBillboard.StudsOffset = Vector3.new(0, 5, 0)
                    saleBillboard.Adornee = muleModel.PrimaryPart
                    saleBillboard.Parent = muleModel.PrimaryPart
                    
                    local saleText = Instance.new("TextLabel")
                    saleText.Size = UDim2.new(1, 0, 1, 0)
                    saleText.Text = "💰 Venda realizada!\n+" .. saleValue .. "$"
                    saleText.TextColor3 = Color3.fromRGB(0, 255, 0)
                    saleText.TextScaled = true
                    saleText.BackgroundTransparency = 1
                    saleText.Parent = saleBillboard
                    
                    -- Som de dinheiro
                    local moneySound = Instance.new("Sound")
                    moneySound.SoundId = "rbxassetid://9118820399"
                    moneySound.Volume = 0.5
                    moneySound.Parent = muleModel.PrimaryPart
                    moneySound:Play()
                    
                    game:GetService("Debris"):AddItem(saleBillboard, 3)
                    game:GetService("Debris"):AddItem(moneySound, 4)
                end
            end
        end
    end
    
    isMiningActive = false
    return true, "Mineração concluída!"
end

-- Conectar RemoteEvents
CreateWaypointEvent.OnServerEvent:Connect(function(player, position)
    local waypoint, errorMsg = createWaypoint(position, player)
    if errorMsg then
        -- Enviar notificação ao jogador
    end
end)

StartMiningEvent.OnServerEvent:Connect(function(player, useMuleIndex)
    local success, message = startMiningRoute(player)
    
    -- Notificar jogador
    if not success then
        warn("Erro na mineração: " .. message)
    end
end)

DeletePathEvent.OnServerEvent:Connect(function(player)
    -- Limpar waypoints deste jogador
    for i = #waypoints, 1, -1 do
        if waypoints[i].Owner == player then
            if waypoints[i].Part and waypoints[i].Part.Parent then
                waypoints[i].Part:Destroy()
            end
            table.remove(waypoints, i)
        end
    end
    
    -- Limpar partes do caminho
    for _, part in ipairs(pathParts) do
        if part and part.Parent then
            part:Destroy()
        end
    end
    
    pathParts = {}
end)

BuyMuleEvent.OnServerEvent:Connect(function(player)
    local playerData = PlayerInventory[player.UserId]
    
    if playerData.Money >= CONFIG.MulePrice then
        playerData.Money = playerData.Money - CONFIG.MulePrice
        playerData.Mules = playerData.Mules + 1
        
        -- Atualizar GUI
        local moneyStat = player.leaderstats:FindFirstChild("Money")
        local mulesStat = player.leaderstats:FindFirstChild("Mules")
        
        if moneyStat then moneyStat.Value = playerData.Money end
        if mulesStat then mulesStat.Value = playerData.Mules end
        
        return true, "Mula comprada com sucesso!"
    else
        return false, "Dinheiro insuficiente!"
    end
end)

-- Inicializar jogadores
game.Players.PlayerAdded:Connect(function(player)
    setupPlayerData(player)
    
    -- Limpar ao sair
    player.CharacterRemoving:Connect(function()
        DeletePathEvent:FireClient(player)
    end)
end)

-- Limpeza global
game.Players.PlayerRemoving:Connect(function(player)
    DeletePathEvent:FireClient(player)
    PlayerInventory[player.UserId] = nil
end)

-- Serviço de salvamento (opcional)
local DataStoreService = game:GetService("DataStoreService")
local miningDataStore = DataStoreService:GetDataStore("TWW_MiningData")

game:BindToClose(function()
    for userId, data in pairs(PlayerInventory) do
        pcall(function()
            miningDataStore:SetAsync(tostring(userId), data)
        end)
    end
end)
-- Cliente: TWWMiningGUI.client.lua
-- Interface estilo The Wild West

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Remotes
local RemoteEvents = ReplicatedStorage:WaitForChild("TWWRemoteEvents")
local CreateWaypointEvent = RemoteEvents:WaitForChild("CreateWaypoint")
local StartMiningEvent = RemoteEvents:WaitForChild("StartMining")
local DeletePathEvent = RemoteEvents:WaitForChild("DeletePath")
local BuyMuleEvent = RemoteEvents:WaitForChild("BuyMule")

-- Criar GUI no estilo western
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "TWW_MiningGUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

-- Frame principal (estilo madeira)
local mainFrame = Instance.new("Frame")
mainFrame.Name = "MiningPanel"
mainFrame.Size = UDim2.new(0, 350, 0, 400)
mainFrame.Position = UDim2.new(1, -360, 0.5, -200)
mainFrame.AnchorPoint = Vector2.new(0, 0.5)
mainFrame.BackgroundColor3 = Color3.fromRGB(60, 40, 20)
mainFrame.BackgroundTransparency = 0.1
mainFrame.BorderColor3 = Color3.fromRGB(100, 70, 30)
mainFrame.BorderSizePixel = 3
mainFrame.Parent = screenGui

-- Título estilo western
local title = Instance.new("TextLabel")
title.Name = "Title"
title.Text = "SISTEMA DE MINERAÇÃO"
title.Size = UDim2.new(1, 0, 0, 40)
title.Position = UDim2.new(0, 0, 0, 0)
title.BackgroundColor3 = Color3.fromRGB(40, 20, 10)
title.TextColor3 = Color3.fromRGB(255, 215, 0)
title.Font = Enum.Font.Fantasy
title.TextSize = 20
title.TextStrokeTransparency = 0.5
title.Parent = mainFrame

-- Subtítulo
local subtitle = Instance.new("TextLabel")
subtitle.Name = "Subtitle"
subtitle.Text = "Prospecção Automatizada"
subtitle.Size = UDim2.new(1, 0, 0, 25)
subtitle.Position = UDim2.new(0, 0, 0, 40)
subtitle.BackgroundTransparency = 1
subtitle.TextColor3 = Color3.fromRGB(200, 200, 200)
subtitle.Font = Enum.Font.SourceSans
subtitle.TextSize = 16
subtitle.Parent = mainFrame

-- Seção: Ferramentas
local toolsSection = Instance.new("Frame")
toolsSection.Name = "ToolsSection"
toolsSection.Size = UDim2.new(0.9, 0, 0, 100)
toolsSection.Position = UDim2.new(0.05, 0, 0.2, 0)
toolsSection.BackgroundColor3 = Color3.fromRGB(80, 60, 40)
toolsSection.BackgroundTransparency = 0.3
toolsSection.BorderSizePixel = 0
toolsSection.Parent = mainFrame

local toolsLabel = Instance.new("TextLabel")
toolsLabel.Name = "ToolsLabel"
toolsLabel.Text = "FERRAMENTAS"
toolsLabel.Size = UDim2.new(1, 0, 0, 25)
toolsLabel.BackgroundColor3 = Color3.fromRGB(60, 40, 20)
toolsLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
toolsLabel.Font = Enum.Font.SourceSansBold
toolsLabel.TextSize = 14
toolsLabel.Parent = toolsSection

-- Botão de waypoints
local waypointButton = Instance.new("TextButton")
waypointButton.Name = "WaypointButton"
waypointButton.Text = "📍 COLOCAR MARCADORES (M)"
waypointButton.Size = UDim2.new(1, -10, 0, 30)
waypointButton.Position = UDim2.new(0, 5, 0, 30)
waypointButton.BackgroundColor3 = Color3.fromRGB(100, 70, 30)
waypointButton.TextColor3 = Color3.fromRGB(255, 255, 255)
waypointButton.Font = Enum.Font.SourceSansBold
waypointButton.TextSize = 14
waypointButton.Parent = toolsSection

-- Botão de limpar
local clearButton = Instance.new("TextButton")
clearButton.Name = "ClearButton"
clearButton.Text = "🗑️ LIMPAR ROTA (R)"
clearButton.Size = UDim2.new(1, -10, 0, 30)
clearButton.Position = UDim2.new(0, 5, 0, 65)
clearButton.BackgroundColor3 = Color3.fromRGB(120, 40, 40)
clearButton.TextColor3 = Color3.fromRGB(255, 255, 255)
clearButton.Font = Enum.Font.SourceSans
clearButton.TextSize = 14
clearButton.Parent = toolsSection

-- Seção: Mulas
local mulesSection = Instance.new("Frame")
mulesSection.Name = "MulesSection"
mulesSection.Size = UDim2.new(0.9, 0, 0, 120)
mulesSection.Position = UDim2.new(0.05, 0, 0.5, 0)
mulesSection.BackgroundColor3 = Color3.fromRGB(80, 60, 40)
mulesSection.BackgroundTransparency = 0.3
mulesSection.BorderSizePixel = 0
mulesSection.Parent = mainFrame

local mulesLabel = Instance.new("TextLabel")
mulesLabel.Name = "MulesLabel"
mulesLabel.Text = "MULAS DE CARGA"
mulesLabel.Size = UDim2.new(1, 0, 0, 25)
mulesLabel.BackgroundColor3 = Color3.fromRGB(60, 40, 20)
mulesLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
mulesLabel.Font = Enum.Font.SourceSansBold
mulesLabel.TextSize = 14
mulesLabel.Parent = mulesSection

-- Botão comprar mula
local buyMuleButton = Instance.new("TextButton")
buyMuleButton.Name = "BuyMuleButton"
buyMuleButton.Text = "🐴 COMPRAR MULA ($50)"
buyMuleButton.Size = UDim2.new(1, -10, 0, 30)
buyMuleButton.Position = UDim2.new(0, 5, 0, 30)
buyMuleButton.BackgroundColor3 = Color3.fromRGB(60, 100, 40)
buyMuleButton.TextColor3 = Color3.fromRGB(255, 255, 255)
buyMuleButton.Font = Enum.Font.SourceSansBold
buyMuleButton.TextSize = 14
buyMuleButton.Parent = mulesSection

-- Botão iniciar mineração
local mineButton = Instance.new("TextButton")
mineButton.Name = "MineButton"
mineButton.Text = "⛏️ INICIAR MINERAÇÃO (E)"
mineButton.Size = UDim2.new(1, -10, 0, 30)
mineButton.Position = UDim2.new(0, 5, 0, 65)
mineButton.BackgroundColor3 = Color3.fromRGB(150, 100, 30)
mineButton.TextColor3 = Color3.fromRGB(255, 255, 255)
mineButton.Font = Enum.Font.SourceSansBold
mineButton.TextSize = 14
mineButton.Parent = mulesSection

-- Display de status
local statusDisplay = Instance.new("Frame")
statusDisplay.Name = "StatusDisplay"
statusDisplay.Size = UDim2.new(0.9, 0, 0, 80)
statusDisplay.Position = UDim2.new(0.05, 0, 0.85, 0)
statusDisplay.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
statusDisplay.BackgroundTransparency = 0.5
statusDisplay.BorderSizePixel = 0
statusDisplay.Parent = mainFrame

local waypointCounter = Instance.new("TextLabel")
waypointCounter.Name = "WaypointCounter"
waypointCounter.Text = "Marcadores: 0/15"
waypointCounter.Size = UDim2.new(1, 0, 0.5, 0)
waypointCounter.Position = UDim2.new(0, 5, 0, 0)
waypointCounter.TextColor3 = Color3.fromRGB(200, 200, 200)
waypointCounter.BackgroundTransparency = 1
waypointCounter.Font = Enum.Font.SourceSans
waypointCounter.TextSize = 14
waypointCounter.TextXAlignment = Enum.TextXAlignment.Left
waypointCounter.Parent = statusDisplay

local loadCounter = Instance.new("TextLabel")
loadCounter.Name = "LoadCounter"
loadCounter.Text = "Carga: 0/50 lbs"
loadCounter.Size = UDim2.new(1, 0, 0.5, 0)
loadCounter.Position = UDim2.new(0, 5, 0.5, 0)
loadCounter.TextColor3 = Color3.fromRGB(200, 200, 200)
loadCounter.BackgroundTransparency = 1
loadCounter.Font = Enum.Font.SourceSans
loadCounter.TextSize = 14
loadCounter.TextXAlignment = Enum.TextXAlignment.Left
loadCounter.Parent = statusDisplay

-- Variáveis
local isPlacingMarkers = false
local mouse = player:GetMouse()

-- Função para obter posição no terreno
local function getGroundPosition()
    local rayOrigin = mouse.Origin
    local rayDirection = mouse.UnitRay.Direction * 500
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    raycastParams.FilterDescendantsInstances = {player.Character}
    
    local raycastResult = workspace:Raycast(rayOrigin, rayDirection, raycastParams)
    
    if raycastResult then
        return raycastResult.Position
    end
    
    return mouse.Hit.Position
end

-- Sistema de marcação
waypointButton.MouseButton1Click:Connect(function()
    isPlacingMarkers = not isPlacingMarkers
    
    if isPlacingMarkers then
        waypointButton.BackgroundColor3 = Color3.fromRGB(150, 100, 30)
        waypointButton.Text = "🎯 MARCANDO... (CLIQUE NO CHÃO)"
        
        -- Mostrar mira
        local crosshair = Instance.new("ScreenGui")
        crosshair.Name = "Crosshair"
        crosshair.Parent = playerGui
        
        local center = Instance.new("Frame")
        center.Size = UDim2.new(0, 10, 0, 10)
        center.Position = UDim2.new(0.5, -5, 0.5, -5)
        center.BackgroundColor3 = Color3.fromRGB(255, 215, 0)
        center.BorderSizePixel = 0
        center.Parent = crosshair
        
        -- Conexão para cliques
        local connection
        connection = mouse.Button1Down:Connect(function()
            if isPlacingMarkers then
                local position = getGroundPosition()
                CreateWaypointEvent:FireServer(position)
                
                -- Feedback sonoro
                local sound = Instance.new("Sound")
                sound.SoundId = "rbxassetid://140691535"
                sound.Volume = 0.3
                sound.Parent = game.Workspace
                sound:Play()
                game:GetService("Debris"):AddItem(sound, 3)
            end
        end)
        
        -- Atualizar enquanto coloca marcadores
        while isPlacingMarkers do
            waypointButton.Text = "🎯 MARCANDO... " .. math.random(1,3) .. " CLIQUE NO CHÃO"
            task.wait(0.5)
        end
        
        connection:Disconnect()
        crosshair:Destroy()
    else
        waypointButton.BackgroundColor3 = Color3.fromRGB(100, 70, 30)
        waypointButton.Text = "📍 COLOCAR MARCADORES (M)"
    end
end)

-- Iniciar mineração
mineButton.MouseButton1Click:Connect(function()
    StartMiningEvent:FireServer()
    
    -- Feedback visual
    mineButton.BackgroundColor3 = Color3.fromRGB(50, 150, 50)
    mineButton.Text = "⛏️ MINERANDO..."
    
    task.wait(2)
    
    mineButton.BackgroundColor3 = Color3.fromRGB(150, 100, 30)
    mineButton.Text = "⛏️ INICIAR MINERAÇÃO (E)"
end)

-- Comprar mula
buyMuleButton.MouseButton1Click:Connect(function()
    BuyMuleEvent:FireServer()
    
    -- Feedback
    buyMuleButton.BackgroundColor3 = Color3.fromRGB(30, 150, 30)
    buyMuleButton.Text = "✅ MULA COMPRADA!"
    
    task.wait(1)
    
    buyMuleButton.BackgroundColor3 = Color3.fromRGB(60, 100, 40)
    buyMuleButton.Text = "🐴 COMPRAR MULA ($50)"
end)

-- Limpar rota
clearButton.MouseButton1Click:Connect(function()
    DeletePathEvent:FireServer()
    
    clearButton.BackgroundColor3 = Color3.fromRGB(200, 50, 50)
    clearButton.Text = "✅ ROTA LIMPA!"
    
    task.wait(1)
    
    clearButton.BackgroundColor3 = Color3.fromRGB(120, 40, 40)
    clearButton.Text = "🗑️ LIMPAR ROTA (R)"
end)

-- Atalhos de teclado
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == Enum.KeyCode.M then
        isPlacingMarkers = not isPlacingMarkers
        waypointButton:Activate()
        
    elseif input.KeyCode == Enum.KeyCode.E then
        mineButton:Activate()
        
    elseif input.KeyCode == Enum.KeyCode.R then
        clearButton:Activate()
        
    elseif input.KeyCode == Enum.KeyCode.B then
        buyMuleButton:Activate()
    end
end)

-- Atualizar contadores (simulado)
local function updateCounters()
    while true do
        task.wait(1)
        
        -- Aqui você conectaria com valores reais do servidor
        -- Por enquanto, apenas placeholder
        waypointCounter.Text = "Marcadores: " .. math.random(0,15) .. "/15"
        loadCounter.Text = "Carga: " .. math.random(0,50) .. "/50 lbs"
    end
end

-- Iniciar atualização
task.spawn(updateCounters)

-- Tooltip flutuante
local tooltip = Instance.new("ScreenGui")
tooltip.Name = "MiningTooltip"
tooltip.Parent = playerGui

local tooltipFrame = Instance.new("Frame")
tooltipFrame.Name = "TooltipFrame"
tooltipFrame.Size = UDim2.new(0, 200, 0, 80)
tooltipFrame.Position = UDim2.new(0.5, -100, 0, 100)
tooltipFrame.BackgroundColor3 = Color3.fromRGB(40, 30, 20)
tooltipFrame.BackgroundTransparency = 0.2
tooltipFrame.BorderSizePixel = 2
tooltipFrame.BorderColor3 = Color3.fromRGB(100, 80, 60)
tooltipFrame.Visible = false
tooltipFrame.Parent = tooltip

local tooltipText = Instance.new("TextLabel")
tooltipText.Name = "TooltipText"
tooltipText.Size = UDim2.new(1, -10, 1, -10)
tooltipText.Position = UDim2.new(0, 5, 0, 5)
tooltipText.Text = "Sistema de Mineração Automática\nUse as teclas M, E, R, B"
tooltipText.TextColor3 = Color3.fromRGB(255, 255, 255)
tooltipText.BackgroundTransparency = 1
tooltipText.Font = Enum.Font.SourceSans
tooltipText.TextSize = 12
tooltipText.TextWrapped = true
tooltipText.Parent = tooltipFrame

-- Mostrar tooltip ao passar mouse
local buttons = {waypointButton, mineButton, clearButton, buyMuleButton}
local tooltips = {
    "Colocar marcadores no chão para definir a rota de mineração",
    "Iniciar processo automático de mineração com sua mula",
    "Limpar todos os marcadores e cancelar operação",
    "Comprar uma mula para carregar mais minérios"
}

for i, button in ipairs(buttons) do
    button.MouseEnter:Connect(function()
        tooltipFrame.Visible = true
        tooltipText.Text = tooltips[i]
    end)
    
    button.MouseLeave:Connect(function()
        tooltipFrame.Visible = false
    end)
end
-- Módulo: TWW_MiningConfig.lua
-- Configurações do sistema de mineração

local Config = {}

-- Preços e valores
Config.Prices = {
    Mule = 50,
    Pickaxe = 25,
    UpgradeCapacity = 100
}

-- Limites
Config.Limits = {
    MaxWaypoints = 15,
    MaxMules = 3,
    MaxLoad = 50,
    MiningRadius = 100
}

-- Experiência
Config.Experience = {
    BaseExpPerOre = 10,
    LevelMultiplier = 1.2,
    ExpForNextLevel = function(level)
        return 100 * (1.5 ^ (level - 1))
    end
}

-- Animações (IDs do Roblox)
Config.Animations = {
    Mining = "rbxassetid://5918726674",
    Walking = "rbxassetid://5915718677",
    Idle = "rbxassetid://5915718677"
}

-- Sons
Config.Sounds = {
    PlaceMarker = "rbxassetid://140691535",
    Mining = "rbxassetid://9081602397",
    Sell = "rbxassetid://9118820399",
    LevelUp = "rbxassetid://278549023"
}

-- Cores (estilo western)
Config.Colors = {
    Waypoint = Color3.fromRGB(139, 69, 19), -- Marrom madeira
    Trail = Color3.fromRGB(160, 120, 80), -- Poeira
    Gold = Color3.fromRGB(255, 215, 0),
    Silver = Color3.fromRGB(192, 192, 192),
    Copper = Color3.fromRGB(184, 115, 51)
}

-- Texturas
Config.Textures = {
    Marker = "rbxassetid://2781140922", -- Pegada
    Trail = "rbxassetid://2809280113", -- Poeira
    Wood = "rbxassetid://2790389807" -- Textura de madeira
}

return Config
