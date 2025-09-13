-- Script único (coloque em ServerScriptService)
-- Nome sugerido: "ExpulsaHub_Script"
-- ⚠️ ATENÇÃO: Substitua `OWNER_NAME` pelo seu nome de jogador (ou use OWNER_USERID com o número)
-- Este script cria o RemoteEvent, escuta pedidos de kick no servidor,
-- e injeta uma GUI apenas para o dono do jogo para digitar o nome e expulsar players.

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- === CONFIGURAÇÃO ===
local OWNER_NAME = "kick hub"        -- <- Troque para o seu nome de jogador Roblox (ou use USERID abaixo)
local OWNER_USERID = nil                -- <- Opcional: coloque o número do seu UserId (ex: 12345678). Se não quiser usar, deixe nil.
local KICK_MESSAGE = "You have been banned for violating the rules." -- Mensagem enviada ao jogador expulso
-- =====================

-- Criar (ou pegar) RemoteEvent em ReplicatedStorage
local kickEvent = ReplicatedStorage:FindFirstChild("ExpulsaHub_KickEvent")
if not kickEvent then
    kickEvent = Instance.new("RemoteEvent")
    kickEvent.Name = "ExpulsaHub_KickEvent"
    kickEvent.Parent = ReplicatedStorage
end

-- Handler no servidor: só aceita pedidos do dono configurado
kickEvent.OnServerEvent:Connect(function(requestingPlayer, targetName)
    -- validação de permissão
    local allowed = false
    if OWNER_USERID and type(OWNER_USERID) == "number" then
        allowed = (requestingPlayer.UserId == OWNER_USERID)
    else
        allowed = (requestingPlayer.Name == OWNER_NAME)
    end

    if not allowed then
        -- Tentativa não autorizada: opcionalmente desconectar quem tentou
        -- requestingPlayer:Kick("Tentativa de usar Expulsa Hub sem permissão.")
        return
    end

    if type(targetName) ~= "string" then return end
    targetName = targetName:match("^%s*(.-)%s*$") -- trim

    if targetName == "" then return end

    local target = Players:FindFirstChild(targetName)
    if target and target ~= requestingPlayer then
        -- Expulsa o jogador alvo
        target:Kick(KICK_MESSAGE)
    end
end)

-- Função que injeta a GUI (com LocalScript interno) no PlayerGui do dono quando entrar
local function giveOwnerGui(player)
    -- valida se é o dono
    local isOwner = false
    if OWNER_USERID and type(OWNER_USERID) == "number" then
        isOwner = (player.UserId == OWNER_USERID)
    else
        isOwner = (player.Name == OWNER_NAME)
    end
    if not isOwner then return end

    -- Criar ScreenGui
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "ExpulsaHub"
    screenGui.ResetOnSpawn = false

    -- Frame principal (estético simples; ajuste à vontade)
    local frame = Instance.new("Frame")
    frame.Name = "Main"
    frame.Size = UDim2.new(0, 300, 0, 120)
    frame.Position = UDim2.new(0.5, -150, 0.1, 0)
    frame.AnchorPoint = Vector2.new(0.5, 0)
    frame.BackgroundColor3 = Color3.fromRGB(30,30,30)
    frame.BorderSizePixel = 0
    frame.Parent = screenGui

    -- Título
    local title = Instance.new("TextLabel")
    title.Name = "Title"
    title.Size = UDim2.new(1, 0, 0, 28)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "Expulsa Hub"
    title.TextColor3 = Color3.fromRGB(255,255,255)
    title.Font = Enum.Font.SourceSansBold
    title.TextSize = 20
    title.Parent = frame

    -- TextBox para digitar nome
    local textBox = Instance.new("TextBox")
    textBox.Name = "TargetNameBox"
    textBox.Size = UDim2.new(1, -20, 0, 36)
    textBox.Position = UDim2.new(0, 10, 0, 36)
    textBox.PlaceholderText = "Digite o nome do jogador..."
    textBox.ClearTextOnFocus = false
    textBox.Font = Enum.Font.SourceSans
    textBox.TextSize = 18
    textBox.Text = ""
    textBox.Parent = frame

    -- Botão de Kick
    local kickButton = Instance.new("TextButton")
    kickButton.Name = "KickButton"
    kickButton.Size = UDim2.new(0.5, -15, 0, 32)
    kickButton.Position = UDim2.new(0.5, 5, 0, 78)
    kickButton.AnchorPoint = Vector2.new(0, 0)
    kickButton.Text = "Kick"
    kickButton.Font = Enum.Font.SourceSansBold
    kickButton.TextSize = 18
    kickButton.Parent = frame

    -- Botão de fechar
    local closeButton = Instance.new("TextButton")
    closeButton.Name = "CloseButton"
    closeButton.Size = UDim2.new(0.5, -15, 0, 32)
    closeButton.Position = UDim2.new(0, 10, 0, 78)
    closeButton.Text = "Fechar"
    closeButton.Font = Enum.Font.SourceSans
    closeButton.TextSize = 18
    closeButton.Parent = frame

    -- LocalScript que roda no cliente (injeta o comportamento dos botões)
    local localScript = Instance.new("LocalScript")
    localScript.Name = "ExpulsaHubClient"

    -- O código do LocalScript como string (o servidor escreve o código aqui)
    localScript.Source = [[
        local ReplicatedStorage = game:GetService("ReplicatedStorage")
        local Players = game:GetService("Players")
        local player = Players.LocalPlayer

        local kickEvent = ReplicatedStorage:WaitForChild("ExpulsaHub_KickEvent")

        local screenGui = script.Parent
        local frame = screenGui:WaitForChild("Main")
        local textBox = frame:WaitForChild("TargetNameBox")
        local kickButton = frame:WaitForChild("KickButton")
        local closeButton = frame:WaitForChild("CloseButton")

        -- Enviar pedido de kick ao servidor
        local function tryKick()
            local name = tostring(textBox.Text or "")
            if name ~= "" then
                kickEvent:FireServer(name)
            end
        end

        kickButton.MouseButton1Click:Connect(tryKick)
        -- Suporta Enter na TextBox
        textBox.FocusLost:Connect(function(enterPressed)
            if enterPressed then
                tryKick()
            end
        end)

        closeButton.MouseButton1Click:Connect(function()
            screenGui:Destroy()
        end)
    ]]

    -- Parentar a GUI ao PlayerGui (LocalScript começará a rodar)
    screenGui.Parent = player:WaitForChild("PlayerGui")
    localScript.Parent = screenGui
end

-- Dar GUI ao dono ao entrar (ou reentrar)
Players.PlayerAdded:Connect(function(player)
    -- espera PlayerGui existir
    player.CharacterAdded:Wait() -- garante que jogador carregou (opcional)
    -- pequena espera para PlayerGui existir
    if not player:FindFirstChild("PlayerGui") then
        player:WaitForChild("PlayerGui", 5)
    end
    pcall(function() giveOwnerGui(player) end)
end)

-- Se o dono já estiver online quando o script rodar, garante a GUI
for _, p in pairs(Players:GetPlayers()) do
    pcall(function() giveOwnerGui(p) end)
end
