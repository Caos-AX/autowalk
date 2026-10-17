
--========================================================
-- AUTO WALK ADVANCED ULTIMATE
-- Para Roblox Studio: LocalScript
-- Coloque em StarterPlayer > StarterPlayerScripts
--
-- Recursos:
-- 1. Desvio de obstaculos
-- 2. Pulos inteligentes
-- 3. Recuperacao contra travamento
-- 4. Variacao natural de velocidade e pausas
-- 5. Exploracao em varias direcoes
-- 6. Verificacao de terreno e quedas
-- 7. Retomada apos respawn
-- 8. Controle de velocidade
-- 9. Interface arrastavel
-- 10. Exploracao continua
--========================================================

local Players = game:GetService("Players")
local PathfindingService = game:GetService("PathfindingService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local random = Random.new()

-- CONFIGURACOES
local DEFAULT_SPEED = 16
local MIN_SPEED = 8
local MAX_SPEED = 24

local MIN_DISTANCE = 20
local MAX_DISTANCE = 65
local CHECK_DISTANCE = 7
local STUCK_TIME = 2.5
local MAX_ROUTE_RETRIES = 3
local MAX_HISTORY = 12

-- ESTADO
local enabled = false
local character
local humanoid
local rootPart
local currentSpeed = DEFAULT_SPEED
local loopRunning = false
local currentStatus = "Desativado"
local destinationHistory = {}
local lastPosition
local stuckSince = 0
local characterConnections = {}

--========================================================
-- INTERFACE
--========================================================

local gui = Instance.new("ScreenGui")
gui.Name = "AutoWalkUltimate"
gui.ResetOnSpawn = false
gui.DisplayOrder = 20

local parentSuccess = pcall(function()
    gui.Parent = player:WaitForChild("PlayerGui")
end)

if not parentSuccess then
    warn("Nao foi possivel criar a interface.")
    return
end

local frame = Instance.new("Frame")
frame.Name = "Main"
frame.Size = UDim2.fromOffset(280, 270)
frame.Position = UDim2.new(0, 30, 0.35, 0)
frame.BackgroundColor3 = Color3.fromRGB(28, 31, 39)
frame.BorderSizePixel = 0
frame.Active = true
frame.Parent = gui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 12)
frameCorner.Parent = frame

local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(75, 85, 105)
stroke.Thickness = 1
stroke.Parent = frame

local function makeLabel(name, text, size, position, fontSize)
    local label = Instance.new("TextLabel")
    label.Name = name
    label.Size = size
    label.Position = position
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(240, 240, 245)
    label.Font = Enum.Font.Gotham
    label.TextSize = fontSize
    label.Parent = frame
    return label
end

local function makeButton(name, text, size, position, color)
    local button = Instance.new("TextButton")
    button.Name = name
    button.Size = size
    button.Position = position
    button.BackgroundColor3 = color
    button.Text = text
    button.TextColor3 = Color3.new(1, 1, 1)
    button.Font = Enum.Font.GothamBold
    button.TextSize = 13
    button.AutoButtonColor = true
    button.Parent = frame

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 7)
    corner.Parent = button

    return button
end

makeLabel(
    "Title",
    "AUTO WALK ULTIMATE",
    UDim2.new(1, -20, 0, 30),
    UDim2.fromOffset(10, 7),
    15
).Font = Enum.Font.GothamBold

local statusLabel = makeLabel(
    "Status",
    "Status: Desativado",
    UDim2.new(1, -20, 0, 24),
    UDim2.fromOffset(10, 39),
    12
)

statusLabel.TextColor3 = Color3.fromRGB(190, 205, 220)

local toggleButton = makeButton(
    "Toggle",
    "ATIVAR CAMINHADA",
    UDim2.new(1, -24, 0, 36),
    UDim2.fromOffset(12, 67),
    Color3.fromRGB(35, 155, 90)
)

local speedLabel = makeLabel(
    "SpeedLabel",
    "Velocidade: 16",
    UDim2.new(1, -24, 0, 23),
    UDim2.fromOffset(12, 112),
    12
)

local minusButton = makeButton(
    "SpeedDown",
    "-",
    UDim2.fromOffset(42, 30),
    UDim2.fromOffset(12, 140),
    Color3.fromRGB(65, 75, 95)
)

local plusButton = makeButton(
    "SpeedUp",
    "+",
    UDim2.fromOffset(42, 30),
    UDim2.fromOffset(226, 140),
    Color3.fromRGB(65, 75, 95)
)

local speedBar = Instance.new("Frame")
speedBar.Size = UDim2.fromOffset(150, 8)
speedBar.Position = UDim2.fromOffset(65, 151)
speedBar.BackgroundColor3 = Color3.fromRGB(65, 70, 82)
speedBar.BorderSizePixel = 0
speedBar.Parent = frame

Instance.new("UICorner", speedBar).CornerRadius = UDim.new(1, 0)

local speedFill = Instance.new("Frame")
speedFill.Size = UDim2.new(
    (currentSpeed - MIN_SPEED) / (MAX_SPEED - MIN_SPEED),
    0, 1, 0
)
speedFill.BackgroundColor3 = Color3.fromRGB(65, 190, 125)
speedFill.BorderSizePixel = 0
speedFill.Parent = speedBar

Instance.new("UICorner", speedFill).CornerRadius = UDim.new(1, 0)

local exploreButton = makeButton(
    "Explore",
    "Exploracao: CONTINUA",
    UDim2.new(1, -24, 0, 32),
    UDim2.fromOffset(12, 183),
    Color3.fromRGB(60, 90, 145)
)

local infoLabel = makeLabel(
    "Info",
    "Desvio | Pulo | Anti-travamento",
    UDim2.new(1, -20, 0, 20),
    UDim2.fromOffset(10, 222),
    10
)
infoLabel.TextColor3 = Color3.fromRGB(165, 180, 200)

makeLabel(
    "Hint",
    "Arraste pelo topo para mover o painel",
    UDim2.new(1, -20, 0, 16),
    UDim2.fromOffset(10, 245),
    9
).TextColor3 = Color3.fromRGB(130, 140, 155)

--========================================================
-- PAINEL ARRASTAVEL
--========================================================

local dragging = false
local dragStart
local frameStart
local dragInput

frame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        and input.Position.Y <= frame.AbsolutePosition.Y + 38 then

        dragging = true
        dragStart = input.Position
        frameStart = frame.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

frame.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and input == dragInput then
        local delta = input.Position - dragStart

        frame.Position = UDim2.new(
            frameStart.X.Scale,
            frameStart.X.Offset + delta.X,
            frameStart.Y.Scale,
            frameStart.Y.Offset + delta.Y
        )
    end
end)

--========================================================
-- PERSONAGEM
--========================================================

local function refreshCharacter()
    character = player.Character

    if not character then
        return false
    end

    humanoid = character:FindFirstChildOfClass("Humanoid")
    rootPart = character:FindFirstChild("HumanoidRootPart")

    return humanoid ~= nil
        and rootPart ~= nil
        and humanoid.Health > 0
end

local function setStatus(message)
    currentStatus = message
    statusLabel.Text = "Status: " .. message
end

local function makeRayParams()
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {character}
    params.IgnoreWater = true
    return params
end

local function raycast(origin, direction, distance)
    if direction.Magnitude < 0.001 then
        return nil
    end

    return Workspace:Raycast(
        origin,
        direction.Unit * distance,
        makeRayParams()
    )
end

--========================================================
-- TERRENO E QUEDAS
--========================================================

local function hasGround(position)
    local origin = position + Vector3.new(0, 3, 0)
    local result = raycast(
        origin,
        Vector3.new(0, -1, 0),
        12
    )

    return result ~= nil
end

local function safeDestination(position)
    if not refreshCharacter() then
        return false
    end

    -- Verifica se existe chao no destino.
    if not hasGround(position) then
        return false
    end

    -- Evita destinos com queda vertical muito grande.
    local heightDifference = position.Y - rootPart.Position.Y

    if heightDifference < -7 then
        return false
    end

    return true
end

--========================================================
-- HISTORICO DE EXPLORACAO
--========================================================

local function wasRecentlyVisited(position)
    for _, previous in ipairs(destinationHistory) do
        if (previous - position).Magnitude < 14 then
            return true
        end
    end

    return false
end

local function rememberDestination(position)
    table.insert(destinationHistory, position)

    while #destinationHistory > MAX_HISTORY do
        table.remove(destinationHistory, 1)
    end
end

--========================================================
-- DETECCAO DE OBSTACULOS
--========================================================

local function chooseDirection()
    if not refreshCharacter() then
        return nil
    end

    local forward = rootPart.CFrame.LookVector
    local right = rootPart.CFrame.RightVector
    local origin = rootPart.Position + Vector3.new(0, 2, 0)

    local frontHit = raycast(origin, forward, CHECK_DISTANCE)

    if not frontHit then
        return forward
    end

    -- Tenta identificar se o obstaculo e baixo.
    local lowOrigin = rootPart.Position + Vector3.new(0, 0.5, 0)
    local lowHit = raycast(lowOrigin, forward, 3)

    if lowHit and humanoid.FloorMaterial ~= Enum.Material.Air then
        humanoid.Jump = true
    end

    -- Procura um lado livre.
    local leftHit = raycast(origin, -right, CHECK_DISTANCE)
    local rightHit = raycast(origin, right, CHECK_DISTANCE)

    if not leftHit and rightHit then
        return -right
    elseif not rightHit and leftHit then
        return right
    elseif not leftHit and not rightHit then
        return random:NextInteger(0, 1) == 0 and right or -right
    end

    -- Ambos os lados bloqueados: recua.
    return -forward
end

--========================================================
-- ESCOLHA DE DESTINOS
--========================================================

local function randomDestination()
    if not refreshCharacter() then
        return nil
    end

    local origin = rootPart.Position

    for attempt = 1, 18 do
        local angle = random:NextNumber(0, math.pi * 2)
        local distance = random:NextNumber(MIN_DISTANCE, MAX_DISTANCE)

        local offset = Vector3.new(
            math.cos(angle) * distance,
            0,
            math.sin(angle) * distance
        )

        local candidate = origin + offset

        -- Procura o chao na posicao candidata.
        local ground = raycast(
            candidate + Vector3.new(0, 30, 0),
            Vector3.new(0, -1, 0),
            100
        )

        if ground then
            candidate = Vector3.new(
                candidate.X,
                ground.Position.Y + 2.5,
                candidate.Z
            )

            if safeDestination(candidate)
                and not wasRecentlyVisited(candidate) then

                rememberDestination(candidate)
                return candidate
            end
        end
    end

    -- Fallback: tenta uma direcao com obstaculos verificados.
    local direction = chooseDirection()

    if direction then
        local candidate = origin + direction * MIN_DISTANCE

        if safeDestination(candidate) then
            return candidate
        end
    end

    return nil
end

--========================================================
-- ANTI-TRAVAMENTO
--========================================================

local function recoverMovement()
    if not refreshCharacter() then
        return
    end

    setStatus("Recuperando movimento...")

    local direction = chooseDirection()

    -- Tenta pular quando estiver no chao.
    if humanoid.FloorMaterial ~= Enum.Material.Air then
        humanoid.Jump = true
    end

    -- Recuo breve e depois movimento lateral.
    humanoid:Move(-rootPart.CFrame.LookVector, false)
    task.wait(0.35)

    if not enabled or not refreshCharacter() then
        return
    end

    direction = chooseDirection()

    if direction then
        humanoid:Move(direction, false)
        task.wait(0.45)
    end

    lastPosition = rootPart.Position
    stuckSince = os.clock()
end

--========================================================
-- NAVEGACAO
--========================================================

local function navigateTo(destination)
    if not refreshCharacter() then
        return false
    end

    local path = PathfindingService:CreatePath({
        AgentRadius = 2,
        AgentHeight = 5,
        AgentCanJump = true,
        AgentJumpHeight = 8,
        AgentMaxSlope = 35,
        WaypointSpacing = 4
    })

    local computed = pcall(function()
        path:ComputeAsync(rootPart.Position, destination)
    end)

    if not computed or path.Status ~= Enum.PathStatus.Success then
        return false
    end

    local blocked = false
    local blockConnection = path.Blocked:Connect(function()
        blocked = true
    end)

    for _, waypoint in ipairs(path:GetWaypoints()) do
        if not enabled or not refreshCharacter() then
            blockConnection:Disconnect()
            return false
        end

        if blocked then
            blockConnection:Disconnect()
            return false
        end

        -- Velocidade variavel para movimento menos rigido.
        local variation = random:NextNumber(0.92, 1.06)
        humanoid.WalkSpeed = math.clamp(
            currentSpeed * variation,
            MIN_SPEED,
            MAX_SPEED
        )

        if waypoint.Action == Enum.PathWaypointAction.Jump then
            humanoid.Jump = true
        end

        -- Verifica obstaculos na direcao do waypoint.
        local direction = waypoint.Position - rootPart.Position

        if direction.Magnitude > 0.5 then
            local hit = raycast(
                rootPart.Position + Vector3.new(0, 2, 0),
                direction,
                math.min(direction.Magnitude, CHECK_DISTANCE)
            )

            if hit then
                local alternative = chooseDirection()

                if alternative then
                    humanoid:Move(alternative, false)
                    task.wait(0.25)
                end

                blockConnection:Disconnect()
                return false
            end
        end

        humanoid:MoveTo(waypoint.Position)

        local startedAt = os.clock()
        local initialPosition = rootPart.Position
        local lastProgress = os.clock()
        local reached = false

        while enabled and os.clock() - startedAt < 8 do
            if not refreshCharacter() then
                blockConnection:Disconnect()
                return false
            end

            if blocked then
                blockConnection:Disconnect()
                return false
            end

            local remaining =
                (rootPart.Position - waypoint.Position).Magnitude

            if remaining < 3 then
                reached = true
                break
            end

            -- Identifica se o personagem continua progredindo.
            if (rootPart.Position - initialPosition).Magnitude > 1 then
                initialPosition = rootPart.Position
                lastProgress = os.clock()
            end

            if os.clock() - lastProgress > STUCK_TIME then
                blockConnection:Disconnect()
                return false
            end

            -- Tenta pular se houver uma barreira a frente.
            local moveDirection = humanoid.MoveDirection

            if moveDirection.Magnitude > 0 then
                local wall = raycast(
                    rootPart.Position + Vector3.new(0, 2, 0),
                    moveDirection,
                    2.5
                )

                if wall then
                    if humanoid.FloorMaterial ~= Enum.Material.Air then
                        humanoid.Jump = true
                    end
                end
            end

            task.wait(0.1)
        end

        if not reached then
            blockConnection:Disconnect()
            return false
        end
    end

    blockConnection:Disconnect()
    return enabled
end

--========================================================
-- EXPLORACAO CONTINUA
--========================================================

local function mainLoop()
    if loopRunning then
        return
    end

    loopRunning = true

    while enabled do
        if not refreshCharacter() then
            setStatus("Aguardando personagem...")
            task.wait(0.5)
            continue
        end

        humanoid.WalkSpeed = currentSpeed
        setStatus("Explorando mapa...")

        local destination = randomDestination()

        if destination then
            local success = false

            for attempt = 1, MAX_ROUTE_RETRIES do
                if not enabled then
                    break
                end

                if navigateTo(destination) then
                    success = true
                    break
                end

                if enabled then
                    recoverMovement()
                    task.wait(0.2)

                    -- Recalcula destino apos falha.
                    destination = randomDestination()

                    if not destination then
                        break
                    end
                end
            end

            if success and enabled then
                -- Pequenas pausas ocasionais.
                if random:NextNumber() < 0.18 then
                    setStatus("Pausa breve...")
                    task.wait(random:NextNumber(0.2, 0.7))
                end
            end
        else
            -- Se nao houver destino seguro, gira para procurar.
            setStatus("Procurando rota segura...")

            local direction = chooseDirection()

            if direction and refreshCharacter() then
                humanoid:Move(direction, false)
                task.wait(0.4)
            else
                task.wait(0.5)
            end
        end

        task.wait(0.1)
    end

    if refreshCharacter() then
        humanoid:Move(Vector3.zero, false)
        humanoid.WalkSpeed = currentSpeed
    end

    loopRunning = false
    setStatus("Desativado")
end

--========================================================
-- CONTROLES
--========================================================

local function updateSpeed()
    speedLabel.Text = "Velocidade: " .. tostring(currentSpeed)

    speedFill.Size = UDim2.new(
        (currentSpeed - MIN_SPEED) / (MAX_SPEED - MIN_SPEED),
        0, 1, 0
    )

    if refreshCharacter() then
        humanoid.WalkSpeed = currentSpeed
    end
end

minusButton.MouseButton1Click:Connect(function()
    currentSpeed = math.max(MIN_SPEED, currentSpeed - 2)
    updateSpeed()
end)

plusButton.MouseButton1Click:Connect(function()
    currentSpeed = math.min(MAX_SPEED, currentSpeed + 2)
    updateSpeed()
end)

exploreButton.MouseButton1Click:Connect(function()
    MAX_DISTANCE = MAX_DISTANCE == 65 and 100 or 65
    MIN_DISTANCE = MAX_DISTANCE == 100 and 35 or 20

    exploreButton.Text = MAX_DISTANCE == 100
        and "Exploracao: AMPLA"
        or "Exploracao: CONTINUA"
end)

toggleButton.MouseButton1Click:Connect(function()
    enabled = not enabled

    if enabled then
        toggleButton.Text = "DESATIVAR CAMINHADA"
        toggleButton.BackgroundColor3 = Color3.fromRGB(190, 55, 60)
        setStatus("Iniciando...")
        task.spawn(mainLoop)
    else
        toggleButton.Text = "ATIVAR CAMINHADA"
        toggleButton.BackgroundColor3 = Color3.fromRGB(35, 155, 90)
        setStatus("Desativando...")

        if refreshCharacter() then
            humanoid:Move(Vector3.zero, false)
            humanoid:MoveTo(rootPart.Position)
        end
    end
end)

--========================================================
-- RESPAWN
--========================================================

player.CharacterAdded:Connect(function(newCharacter)
    character = newCharacter

    task.wait(1)

    if enabled then
        -- O loop principal detecta o novo personagem.
        refreshCharacter()
        setStatus("Retomando apos respawn...")

        if not loopRunning then
            task.spawn(mainLoop)
        end
    end
end)

player.CharacterRemoving:Connect(function()
    character = nil
    humanoid = nil
    rootPart = nil
end)

-- Inicializacao
refreshCharacter()
updateSpeed()
setStatus("Desativado")
