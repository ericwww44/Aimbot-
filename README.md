-- NPC_Shooter.lua
-- Coloque este Script dentro do modelo do NPC (ex: Workspace.Enemy)
-- Requisitos no modelo do NPC:
--  - Um Part chamado "Head" ou "HumanoidRootPart" (usado como origem do cálculo)
--  - Uma Part chamada "Gun" com um Attachment/Part chamado "FirePoint" (ponto de disparo)
--  - Um Humanoid para a saúde/movimento (opcional)
-- Ajuste os nomes se seu modelo difere.

local NPC = script.Parent
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local Debris = game:GetService("Debris")

-- CONFIGURAÇÃO
local CONFIG = {
    Range = 120,               -- alcance máximo do NPC (studs)
    FireRate = 0.6,            -- segundos entre tiros
    ProjectileSpeed = 200,     -- velocidade do "projétil" usado para previsão (studs/s)
    Damage = 20,               -- dano por tiro
    LeadTarget = true,         -- usar previsão do movimento do jogador?
    RequireLineOfSight = true, -- precisa ter visão direta para atirar?
    TurnSpeed = 8,             -- velocidade de rotação (quanto maior, mais rápido gira)
    BulletLifetime = 3,        -- tempo de vida visual do projétil (se usar efeito)
}

-- PARTS (ajuste se necessário)
local HRP = NPC:FindFirstChild("HumanoidRootPart") or NPC:FindFirstChild("Torso") or NPC:FindFirstChild("UpperTorso")
local Gun = NPC:FindFirstChild("Gun")
local FirePoint = Gun and (Gun:FindFirstChild("FirePoint") or Gun) -- se não tiver attachment, usa a part Gun

assert(HRP, "HumanoidRootPart/Torso não encontrado no NPC.")
assert(Gun, "Parte 'Gun' não encontrada no NPC.")

-- Estado
local lastFire = 0

-- Função para obter jogador mais próximo dentro do alcance
local function getNearestPlayer()
    local nearestPlayer, nearestChar, nearestHRP, nearestDist = nil, nil, nil, math.huge
    for _, pl in pairs(Players:GetPlayers()) do
        local char = pl.Character
        if char then
            local humanoid = char:FindFirstChildOfClass("Humanoid")
            local hrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso") or char:FindFirstChild("UpperTorso")
            if humanoid and hrp and humanoid.Health > 0 then
                local dist = (hrp.Position - HRP.Position).Magnitude
                if dist < CONFIG.Range and dist < nearestDist then
                    nearestPlayer, nearestChar, nearestHRP, nearestDist = pl, char, hrp, dist
                end
            end
        end
    end
    return nearestPlayer, nearestChar, nearestHRP, nearestDist
end

-- Checa linha de visão usando raycast
local function hasLineOfSight(originPos, targetPos)
    if not CONFIG.RequireLineOfSight then return true end
    local dir = (targetPos - originPos)
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {NPC}
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.IgnoreWater = true
    local result = workspace:Raycast(originPos, dir, params)
    -- se raycast bateu e o objeto não é o jogador, bloqueado
    if result then
        local hitPart = result.Instance
        -- considera que se acertou algo do personagem do jogador, visão livre
        if hitPart and hitPart:IsDescendantOf(targetPart.Parent) then
            return true
        else
            return false
        end
    end
    return true
end

-- Função de previsão (lead) - aproximação simples
local function predictedAimPoint(originPos, targetHRP)
    local targetPos = targetHRP.Position
    if not CONFIG.LeadTarget then return targetPos end

    local targetVel = targetHRP.Velocity or Vector3.new(0,0,0)
    -- tempo estimado até o projétil chegar (dist / speed)
    local dist = (targetPos - originPos).Magnitude
    local travelTime = dist / math.max(1, CONFIG.ProjectileSpeed)
    -- posição projetada
    local leadPos = targetPos + targetVel * travelTime
    return leadPos
end

-- Função para causar dano (server-side)
local function applyDamageIfHit(hitInstance, shooter)
    if not hitInstance then return end
    local humanoid = hitInstance.Parent and hitInstance.Parent:FindFirstChildOfClass("Humanoid")
        or hitInstance:FindFirstChildOfClass("Humanoid")
    if humanoid and humanoid.Health > 0 then
        humanoid:TakeDamage(CONFIG.Damage)
    end
end

-- Efeito visual simples de projétil (part)
local function spawnBulletEffect(origin, direction)
    local b = Instance.new("Part")
    b.Size = Vector3.new(0.2,0.2,0.2)
    b.CanCollide = false
    b.Anchored = true
    b.CFrame = CFrame.new(origin)
    b.Parent = workspace
    -- move com Tween simples
    local distance = CONFIG.Range
    local targetPos = origin + direction.Unit * distance
    local travelTime = math.min(CONFIG.BulletLifetime, distance / math.max(1, CONFIG.ProjectileSpeed))
    local start = tick()
    spawn(function()
        while tick() - start < travelTime do
            local t = (tick() - start) / travelTime
            b.CFrame = CFrame.new(origin:Lerp(targetPos, t))
            wait()
        end
        b:Destroy()
    end)
    Debris:AddItem(b, CONFIG.BulletLifetime + 0.1)
end

-- Função de atirar: faz Raycast instantâneo (hitscan) para simplificar danos
local function shootAt(targetHRP)
    local origin = FirePoint.Position
    local aimPoint = predictedAimPoint(origin, targetHRP)
    local direction = (aimPoint - origin)
    if direction.Magnitude <= 0.1 then return end

    -- Efeito visual (opcional)
    spawnBulletEffect(origin, direction)

    -- Raycast para checar hits
    local params = RaycastParams.new()
    params.FilterDescendantsInstances = {NPC, Gun}
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.IgnoreWater = true
    local result = workspace:Raycast(origin, direction, params)
    if result then
        local hitPart = result.Instance
        -- aplica dano se for personagem
        applyDamageIfHit(hitPart, NPC)
    end
end

-- Rotaciona a arma/NPC em direção ao ponto (com suavização)
local function rotateTowards(point, dt)
    -- Rotaciona a parte Gun para mirar no ponto
    local origin = Gun.Position
    local targetCFrame = CFrame.new(origin, point)
    -- suaviza a rotação
    local current = Gun.CFrame
    local lerped = current:Lerp(targetCFrame, math.clamp(CONFIG.TurnSpeed * dt, 0, 1))
    Gun.CFrame = CFrame.new(Gun.Position) * CFrame.Angles(lerped:ToEulerAnglesXYZ())
    -- faz o corpo do NPC girar no yaw para seguir o alvo (opcional)
    local hrpPos = HRP.Position
    local lookAt = Vector3.new(point.X, hrpPos.Y, point.Z)
    local desired = CFrame.new(hrpPos, lookAt)
    HRP.CFrame = HRP.CFrame:lerp(desired, math.clamp(CONFIG.TurnSpeed * dt, 0, 1))
end

-- Loop principal
spawn(function()
    while true do
        local now = tick()
        local pl, char, targetHRP, dist = getNearestPlayer()
        if pl and char and targetHRP then
            -- prever posição
            local aimPoint = predictedAimPoint(FirePoint.Position, targetHRP)

            -- opcional: checar LOS usando raycast (simples)
            -- Implementação otimizada: raycast entre origem e targetHRP.Position
            local losOk = true
            if CONFIG.RequireLineOfSight then
                local params = RaycastParams.new()
                params.FilterDescendantsInstances = {NPC, Gun}
                params.FilterType = Enum.RaycastFilterType.Blacklist
                params.IgnoreWater = true
                local ray = workspace:Raycast(FirePoint.Position, (targetHRP.Position - FirePoint.Position), params)
                if ray and not ray.Instance:IsDescendantOf(targetHRP.Parent) then
                    losOk = false
                end
            end

            -- roteamento suave
            local dt = wait() or 0.03
            rotateTowards(aimPoint, dt)

            if losOk and now - lastFire >= CONFIG.FireRate then
                lastFire = now
                shootAt(targetHRP)
            end
        else
            -- sem alvo: pausa pequena para economizar CPU
            wait(0.15)
        end
        RunService.Heartbeat:Wait()
    end
end)
