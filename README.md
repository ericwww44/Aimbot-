-- [[ SISTEMA DE SEGURANÇA E ADMINS ]] --
local Players = game:GetService("Players")
local localPlayer = Players.LocalPlayer
local adminsPermitidos = {"marazzitie2", "Vitorgabriel_423"} -- Seus Admins salvos
local acessoGarantido = false

for _, admin in pairs(adminsPermitidos) do
    if localPlayer.Name == admin then
        acessoGarantido = true
        break
    end
end

if not acessoGarantido then
    localPlayer:Kick("Acesso Negado: Você não é um Admin autorizado.")
    return
end

-- [[ INTERFACE MOBILE (ESTILO RED RUBY) ]] --
local ScreenGui = Instance.new("ScreenGui")
local MainFrame = Instance.new("Frame")
local Title = Instance.new("TextLabel")
local FarmButton = Instance.new("TextButton")

ScreenGui.Parent = game.CoreGui
MainFrame.Name = "RedzHub_Mobile"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = Color3.fromRGB(45, 0, 0) -- Vermelho Escuro
MainFrame.Position = UDim2.new(0.3, 0, 0.3, 0)
MainFrame.Size = UDim2.new(0, 200, 0, 150)
MainFrame.Active = true
MainFrame.Draggable = true -- Importante para Mobile

Title.Parent = MainFrame
Title.Text = "REDZ HUB - ADMIN"
Title.Size = UDim2.new(1, 0, 0, 30)
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.BackgroundColor3 = Color3.fromRGB(150, 0, 0)

FarmButton.Parent = MainFrame
FarmButton.Position = UDim2.new(0.1, 0, 0.4, 0)
FarmButton.Size = UDim2.new(0.8, 0, 0.4, 0)
FarmButton.Text = "LIGAR AUTO FARM"
FarmButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)

-- [[ LÓGICA DO AUTO FARM 100% ANTIBAN (TWEEN) ]] --
_G.AutoFarm = false
local TweenService = game:GetService("TweenService")

function irPara(targetCFrame)
    local character = localPlayer.Character
    if character and character:FindFirstChild("HumanoidRootPart") then
        local distancia = (character.HumanoidRootPart.Position - targetCFrame.p).Magnitude
        local info = TweenInfo.new(distancia / 50, Enum.EasingStyle.Linear) -- Velocidade segura (50)
        local tween = TweenService:Create(character.HumanoidRootPart, info, {CFrame = targetCFrame})
        tween:Play()
    end
end

FarmButton.MouseButton1Click:Connect(function()
    _G.AutoFarm = not _G.AutoFarm
    if _G.AutoFarm then
        FarmButton.Text = "FARM: LIGADO"
        FarmButton.BackgroundColor3 = Color3.fromRGB(200, 0, 0)
    else
        FarmButton.Text = "FARM: DESLIGADO"
        FarmButton.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
    end
end)

-- Loop de Ataque e Verificação
spawn(function()
    while true do
        task.wait(0.1)
        if _G.AutoFarm then
            pcall(function()
                -- Lógica de Ataque Automático
                local tool = localPlayer.Character:FindFirstChildOfClass("Tool")
                if tool then 
                    tool:Activate() 
                end
                
                -- Exemplo: Ir para o NPC da Quest (Precisa das IDs da nova atualização)
                -- irPara(game.Workspace.NPCs["QuestNPC"].CFrame)
            end)
        end
    end
end)
