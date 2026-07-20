-- Carrega a interface visual estável (Orion Library)
local OrionLib = loadstring(game:HttpGet(('https://githubusercontent.com')))()
local Window = OrionLib:MakeWindow({Name = "Noob Incremental: Auto-Trial", HidePremium = false, SaveConfig = true, ConfigFolder = "OrionTest"})

-- CONFIGURAÇÕES PADRÃO (Variáveis Globais)
_G.AutoTrial = false
_G.TPSpeed = 0.2         -- Tempo de espera entre os teletransportes (Menor = Mais rápido)
_G.MaxRoom = 10          -- Até qual sala o script deve ir antes de sair
_G.AutoLeave = false     -- Ativa/Desativa o abandono automático
_G.JoinCooldown = 1800   -- Tempo de espera do ciclo em segundos (1800s = 30 minutos)

local player = game.Players.LocalPlayer
local tweenService = game:GetService("TweenService")

-- FUNÇÃO DE TELEPORTE SUAVE (Evita detecção de hack de velocidade por mudança brusca)
local function safeTeleport(targetCFrame)
    local character = player.Character or player.CharacterAdded:Wait()
    local root = character:WaitForChild("HumanoidRootPart")
    if root then
        local tweenInfo = TweenInfo.new(_G.TPSpeed, Enum.EasingStyle.Linear)
        local tween = tweenService:Create(root, tweenInfo, {CFrame = targetCFrame})
        tween:Play()
        tween.Completed:Wait()
    end
end

-- ABA PRINCIPAL DA INTERFACE
local MainTab = Window:MakeTab({
    Name = "Controles",
    Icon = "rbxassetid://4483345998",
    PremiumOnly = false
})

-- Componentes da Interface (Toggles, Sliders e Inputs)
MainTab:AddToggle({
    Name = "Ativar Auto-Trial (A cada 30min)",
    Default = false,
    Callback = function(Value)
        _G.AutoTrial = Value
    end
})

MainTab:AddSlider({
    Name = "Velocidade do Teletransporte",
    Min = 1,
    Max = 10,
    Default = 2,
    Color = Color3.fromRGB(255,255,255),
    Increment = 1,
    ValueName = "Suavidade (Menor é mais rápido)",
    Callback = function(Value)
        _G.TPSpeed = Value / 10 -- Converte para escala decimal (0.1 a 1.0)
    end
})

MainTab:AddInput({
    Name = "Ir até a Sala número:",
    Default = "10",
    Numeric = true,
    Callback = function(Value)
        _G.MaxRoom = tonumber(Value) or 10
    end
})

MainTab:AddToggle({
    Name = "Ativar Auto-Leave na sala selecionada",
    Default = false,
    Callback = function(Value)
        _G.AutoLeave = Value
    end
})

-- SISTEMA ANTI-AFK INTEGRADO
local vu = game:GetService("VirtualUser")
game:GetService("Players").LocalPlayer.Idled:Connect(function()
    vu:Button2Down(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
    task.wait(1)
    vu:Button2Up(Vector2.new(0,0), workspace.CurrentCamera.CFrame)
end)

-- LÓGICA DO AUTO-JOIN E ROTAÇÃO AUTOMÁTICA
spawn(function()
    while true do
        task.wait(1)
        if _G.AutoTrial then
            OrionLib:MakeNotification({Name = "Status", Content = "Iniciando verificação da Trial...", Time = 5})
            
            -- 1. TELEPORTAR ATÉ O NPC OU BOTÃO DE ENTRADA DA TRIAL
            -- Nota: O script procura a estrutura da Trial na Área Principal do mapa
            local trialEntrance = workspace:FindFirstChild("TrialEntrance") or workspace:FindFirstChild("TrialLobby")
            if trialEntrance then
                safeTeleport(trialEntrance.CFrame)
                task.wait(2) -- Tempo de espera padrão antes de clicar/entrar
                
                -- Aciona o evento de proximidade ou clique caso exista no local
                if trialEntrance:FindFirstChildOfClass("ProximityPrompt") then
                    fireproximityprompt(trialEntrance:FindFirstChildOfClass("ProximityPrompt"))
                end
            end
            
            -- 2. LOOP DENTRO DA TRIAL (Limpeza das salas e combate aos Mobs)
            local currentRoom = 1
            while _G.AutoTrial and (not _G.AutoLeave or currentRoom <= _G.MaxRoom) do
                task.wait(0.5)
                
                -- Procura monstros marcados como inimigos específicos da sala atual
                local mobFound = false
                for _, mob in pairs(workspace:GetChildren()) do
                    if mob:FindFirstChild("Humanoid") and mob:FindFirstChild("HumanoidRootPart") and mob.Name ~= player.Name then
                        if mob.Humanoid.Health > 0 then
                            mobFound = true
                            -- Teleporta e ataca
                            safeTeleport(mob.HumanoidRootPart.CFrame * CFrame.new(0, 4, 0))
                            
                            -- Equipa arma do inventário e ataca o mob
                            local char = player.Character
                            if char then
                                local tool = char:FindFirstChildOfClass("Tool") or player.Backpack:FindFirstChildOfClass("Tool")
                                if tool then 
                                    tool.Parent = char
                                    tool:Activate() 
                                end
                            end
                        end
                    end
                end
                
                -- Se não há mais monstros na sala, assume que avançou de sala
                if not mobFound then
                    currentRoom = currentRoom + 1
                    task.wait(1) -- Aguarda transição da animação da porta do jogo
                end
                
                -- Coleta os baús/recompensas gerados na sala
                for _, obj in pairs(workspace:GetChildren()) do
                    if string.match(string.lower(obj.Name), "chest") or string.match(string.lower(obj.Name), "reward") then
                        if obj:IsA("BasePart") or obj:FindFirstChild("HumanoidRootPart") then
                            safeTeleport(obj.CFrame)
                            task.wait(0.3)
                        end
                    end
                end
            end
            
            -- 3. AUTO LEAVE (Se atingir o limite configurado)
            if _G.AutoLeave and currentRoom > _G.MaxRoom then
                OrionLib:MakeNotification({Name = "Meta Atingida", Content = "Saindo da Trial automaticamente!", Time = 5})
                -- Executa o comando do jogo para retornar ao spawn/lobby principal
                local spawnPlatform = workspace:FindFirstChild("SpawnLocation")
                if spawnPlatform then
                    safeTeleport(spawnPlatform.CFrame * CFrame.new(0,3,0))
                end
            end
            
            -- 4. CONTROLE DE TEMPO (Aguarda 30 minutos antes do próximo ciclo)
            OrionLib:MakeNotification({Name = "Ciclo Concluído", Content = "Aguardando 30 minutos para a próxima Trial.", Time = 10})
            task.wait(_G.JoinCooldown)
        end
    end
end)

OrionLib:Init()
