--[[
    Dead Rails Security Module
    Author: [Your Name]
    Description: Server-side anti-exploit system for Roblox games
    GitHub: [Your Repo URL]
    Version: 1.0.0
    License: MIT
--]]

local SecuritySystem = {}
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

-- Configuration
local CONFIG = {
    MAX_WALKSPEED = 32,                   -- studs/second (default Roblox sprint speed)
    MAX_JUMPPOWER = 75,                   -- default Roblox jump power
    MAX_BONUS_PER_MINUTE = 120,           -- adjust based on game mechanics
    TELEPORT_THRESHOLD = 100,             -- studs/frame to detect teleport hacking
    FLY_DETECTION_INTERVAL = 1            -- seconds between fly checks
}

-- Track player statistics
local playerStats = {}

--[[
    Detects abnormal movement (speed hacks, teleports, fly hacks)
    @param player: Player instance
--]]
local function monitorMovement(player)
    local character = player.Character or player.CharacterAdded:Wait()
    local humanoid = character:WaitForChild("Humanoid")
    local rootPart = character:WaitForChild("HumanoidRootPart")
    
    local lastPosition = rootPart.Position
    local lastCheck = os.clock()
    
    -- Speed and teleport detection
    RunService.Heartbeat:Connect(function()
        local now = os.clock()
        local deltaTime = now - lastCheck
        lastCheck = now
        
        local currentPosition = rootPart.Position
        local distance = (currentPosition - lastPosition).Magnitude
        lastPosition = currentPosition
        
        -- Teleport detection
        if distance > CONFIG.TELEPORT_THRESHOLD and deltaTime > 0 then
            local speed = distance / deltaTime
            if speed > CONFIG.MAX_WALKSPEED * 3 then  -- Allow some tolerance
                player:Kick("[Security] Teleport hack detected")
                return
            end
        end
    end)
    
    -- Fly detection
    while true do
        task.wait(CONFIG.FLY_DETECTION_INTERVAL)
        if not character:FindFirstChild("Humanoid") then continue end
        
        -- Check if player is flying without proper animation state
        if humanoid:GetState() == Enum.HumanoidStateType.Freefall and rootPart.Velocity.Y > 0 then
            player:Kick("[Security] Fly hack detected")
            return
        end
    end
end

--[[
    Tracks bonus collection rate
    @param player: Player instance
--]]
local function monitorBonusCollection(player)
    playerStats[player.UserId] = {
        bonusCount = 0,
        lastReset = os.time()
    }
    
    player:GetAttributeChangedSignal("BonusCollected"):Connect(function()
        local stats = playerStats[player.UserId]
        stats.bonusCount = stats.bonusCount + 1
        
        -- Reset counter every minute
        if os.time() - stats.lastReset >= 60 then
            stats.bonusCount = 0
            stats.lastReset = os.time()
        end
        
        -- Bonus collection rate check
        if stats.bonusCount > CONFIG.MAX_BONUS_PER_MINUTE then
            player:Kick("[Security] Abnormal bonus collection detected")
        end
    end)
end

--[[
    Initializes security system for a player
    @param player: Player instance
--]]
function SecuritySystem.initPlayer(player)
    if not player:IsDescendantOf(Players) then return end
    
    monitorMovement(player)
    monitorBonusCollection(player)
    
    -- Humanoid property checks
    player.CharacterAdded:Connect(function(character)
        local humanoid = character:WaitForChild("Humanoid")
        
        -- WalkSpeed check
        humanoid:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
            if humanoid.WalkSpeed > CONFIG.MAX_WALKSPEED then
                player:Kick("[Security] Speed hack detected")
            end
        end)
        
        -- JumpPower check
        humanoid:GetPropertyChangedSignal("JumpPower"):Connect(function()
            if humanoid.JumpPower > CONFIG.MAX_JUMPPOWER then
                player:Kick("[Security] Jump hack detected")
            end
        end)
    end)
end

-- Initialize for all players
Players.PlayerAdded:Connect(SecuritySystem.initPlayer)
for _, player in ipairs(Players:GetPlayers()) do
    SecuritySystem.initPlayer(player)
end

return SecuritySystem
