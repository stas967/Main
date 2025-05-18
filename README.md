local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local Window = Rayfield:CreateWindow({
   Name = "Murder Mystery 2",
   Icon = 0, -- Icon in Topbar. Can use Lucide Icons (string) or Roblox Image (number). 0 to use no icon (default).
   LoadingTitle = "The Vault Scripts",
   LoadingSubtitle = "by Stas()",
   Theme = "Default", -- Check https://docs.sirius.menu/rayfield/configuration/themes

   DisableRayfieldPrompts = false,
   DisableBuildWarnings = false, -- Prevents Rayfield from warning when the script has a version mismatch with the interface

   ConfigurationSaving = {
      Enabled = true,
      FolderName = nil, -- Create a custom folder for your hub/game
      FileName = "Big Hub"
   },

   Discord = {
      Enabled = false, -- Prompt the user to join your Discord server if their executor supports it
      Invite = "noinvitelink", -- The Discord invite code, do not include discord.gg/. E.g. discord.gg/ ABCD would be ABCD
      RememberJoins = true -- Set this to false to make them join the discord every time they load it up
   },

   KeySystem = false, -- Set this to true to use our key system
   KeySettings = {
      Title = "Untitled",
      Subtitle = "Key System",
      Note = "No method of obtaining the key is provided", -- Use this to tell the user how to get a key
      FileName = "Key", -- It is recommended to use something unique as other scripts using Rayfield may overwrite your key file
      SaveKey = true, -- The user's key will be saved, but if you change the key, they will be unable to use your script
      GrabKeyFromSite = false, -- If this is true, set Key below to the RAW site you would like Rayfield to get the key from
      Key = {"Hello"} -- List of keys that will be accepted by the system, can be RAW file links (pastebin, github etc) or simple strings ("hello","key22")
   }
})

local Tab = Window:CreateTab("ESP | ", 4483362458) -- Title, Image

local ToggleESP = Tab:CreateToggle({
    Name = "Player ESP",
    CurrentValue = false,
    Flag = "ESPTracersToggle",
    Callback = function(Value)
        local Players = game:GetService("Players")
        local RunService = game:GetService("RunService")
        local LocalPlayer = Players.LocalPlayer

        local espEnabled = Value
        local connections = {}

        -- Check gun or knife status
        local function getWeaponStatus(player)
            local hasGun, hasKnife = false, false

            local function checkItems(container)
                if not container then return end
                for _, item in pairs(container:GetChildren()) do
                    if item:IsA("Tool") then
                        local name = item.Name:lower()
                        if name:find("gun") then
                            hasGun = true
                        elseif name:find("knife") then
                            hasKnife = true
                        end
                    end
                end
            end

            checkItems(player:FindFirstChild("Backpack"))
            checkItems(player.Character)

            return hasGun, hasKnife
        end

        local function createESP(player)
            local character = player.Character
            if not character then return end

            -- Create highlight for the player
            local highlight = character:FindFirstChild("ESP_Highlight")
            if not highlight then
                highlight = Instance.new("Highlight")
                highlight.Name = "ESP_Highlight"
                highlight.FillTransparency = 0.75
                highlight.OutlineTransparency = 0
                highlight.OutlineColor = Color3.new(1, 1, 1)
                highlight.Adornee = character
                highlight.Parent = character
            end
        end

        -- Apply ESP for local player as well
        local function createSelfESP()
            createESP(LocalPlayer)  -- Apply ESP to the local player's character
        end

        local function updateESPColors()
            for _, player in pairs(Players:GetPlayers()) do
                if player.Character then
                    local highlight = player.Character:FindFirstChild("ESP_Highlight")
                    if highlight then
                        local hasGun, hasKnife = getWeaponStatus(player)
                        if hasGun then
                            highlight.FillColor = Color3.fromRGB(0, 0, 255) -- Blue
                        elseif hasKnife then
                            highlight.FillColor = Color3.fromRGB(255, 0, 0) -- Red
                        else
                            highlight.FillColor = Color3.fromRGB(255, 255, 255) -- White
                        end
                    end
                end
            end
        end

        local function clearAllESP()
            for _, player in pairs(Players:GetPlayers()) do
                if player.Character then
                    local highlight = player.Character:FindFirstChild("ESP_Highlight")
                    if highlight then
                        highlight:Destroy()
                    end
                end
            end
            for _, conn in ipairs(connections) do
                if conn and conn.Disconnect then conn:Disconnect() end
            end
            connections = {}
        end

        -- Add ESP for newly respawned players
        local function onPlayerAdded(player)
            -- When the player respawns
            player.CharacterAdded:Connect(function(character)
                wait(1) -- Wait for a moment to ensure the character is fully loaded
                if espEnabled then
                    createESP(player) -- Re-create ESP when respawned
                end
            end)

            -- Remove ESP when the player dies
            player.CharacterRemoving:Connect(function(character)
                local highlight = character:FindFirstChild("ESP_Highlight")
                if highlight then
                    highlight:Destroy() -- Remove highlight when character dies
                end
            end)

            -- Create ESP for the player if they are already in the game
            if espEnabled then
                createESP(player)
            end
        end

        if espEnabled then
            -- Apply to existing players (including local player)
            for _, player in pairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    createESP(player)
                end
            end

            -- Also apply ESP to the local player
            createSelfESP()

            -- Handle respawning players
            table.insert(connections, Players.PlayerAdded:Connect(onPlayerAdded))

            -- Update color every frame
            table.insert(connections, RunService.RenderStepped:Connect(function()
                if espEnabled then
                    updateESPColors()
                end
            end))

        else
            clearAllESP()
        end
    end
})


local ToggleAimbot = Tab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        local Players = game:GetService("Players")
        local LocalPlayer = Players.LocalPlayer
        local Mouse = LocalPlayer:GetMouse()
        local RunService = game:GetService("RunService")
        local ReplicatedStorage = game:GetService("ReplicatedStorage")
        
        local aimbotEnabled = Value
        local targetPlayer = nil
        local smoothness = 0.1 -- Smoothness factor for the aimbot
        local autoShootEnabled = false -- Default: Auto-shoot is off

        -- Function to get the role of the local player
        local function getRole()
            local role = nil
            -- Assuming you have a way to identify the role (Sheriff, Murderer, Innocent)
            -- Example: If it's stored in a Remote or the player's PlayerData
            -- Here I'll assume a hypothetical RemoteEvent that gives the role.
            local roleRemote = ReplicatedStorage:WaitForChild("PlayerRole") 
            role = roleRemote:InvokeServer(LocalPlayer) 
            return role
        end

        -- Function to find the closest target (for Sheriff or Murderer)
        local function getClosestTarget()
            local closestTarget = nil
            local shortestDistance = math.huge

            for _, player in pairs(Players:GetPlayers()) do
                if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
                    local distance = (player.Character.HumanoidRootPart.Position - LocalPlayer.Character.HumanoidRootPart.Position).magnitude
                    if distance < shortestDistance then
                        closestTarget = player
                        shortestDistance = distance
                    end
                end
            end

            return closestTarget
        end

        -- Function for smooth aiming (lerping the mouse position)
        local function smoothAim(targetPosition)
            local currentPos = Mouse.Hit.p
            local newPos = currentPos:Lerp(targetPosition, smoothness)
            Mousemoverel(newPos.X - currentPos.X, newPos.Y - currentPos.Y)
        end

        -- Auto-shoot toggle
        local ToggleAutoShoot = Tab:CreateToggle({
            Name = "Auto-shoot",
            CurrentValue = false,
            Flag = "AutoShootToggle",
            Callback = function(Value)
                autoShootEnabled = Value
            end
        })

        -- The main aimbot logic
        local function aimbot()
            local role = getRole()
            if role == "Sheriff" or role == "Murderer" then
                -- If the player is Sheriff or Murderer, aim at the closest enemy
                targetPlayer = getClosestTarget()
                if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    local targetPosition = targetPlayer.Character.HumanoidRootPart.Position
                    smoothAim(targetPosition)

                    -- Auto-shoot if enabled and the player is aiming at a target
                    if autoShootEnabled and targetPlayer.Character:FindFirstChild("Head") then
                        local ray = Ray.new(Mouse.Hit.p, (targetPosition - Mouse.Hit.p).unit * 500)
                        local hitPart, hitPosition = game:GetService("Workspace"):FindPartOnRay(ray, LocalPlayer.Character)
                        if hitPart and hitPart.Parent == targetPlayer.Character then
                            -- Trigger auto-shoot logic (assuming a Remote or function for shooting)
                            -- Example:
                            ReplicatedStorage:WaitForChild("ShootRemote"):FireServer() 
                        end
                    end
                end
            else
                -- If the player is Innocent, turn off the aimbot
                targetPlayer = nil
            end
        end

        -- Function to turn off aimbot when toggled off
        local function disableAimbot()
            targetPlayer = nil
        end

        -- Connect the aimbot to the RenderStepped to update every frame
        table.insert(connections, RunService.RenderStepped:Connect(function()
            if aimbotEnabled then
                aimbot()
            else
                disableAimbot()
            end
        end))
    end
})

