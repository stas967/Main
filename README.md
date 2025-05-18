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

local Toggle = Tab:CreateToggle({
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

        if espEnabled then
            -- Apply to existing players (including local player)
            for _, player in pairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    createESP(player)
                end
            end

            -- Also apply ESP to the local player
            createSelfESP()

            -- Update color every frame
            table.insert(connections, RunService.RenderStepped:Connect(function()
                if espEnabled then
                    updateESPColors()
                end
            end))

            -- Handle respawning players
            table.insert(connections, Players.PlayerAdded:Connect(function(player)
                table.insert(connections, player.CharacterAdded:Connect(function()
                    wait(1)
                    if espEnabled then
                        createESP(player)
                    end
                end))
            end))
        else
            clearAllESP()
        end
    end
})


local Toggle = Tab:CreateToggle({
    Name = "Smooth Aimbot (Sheriff & Murderer)",
    CurrentValue = false,
    Flag = "SmoothRoleAimbotToggle",
    Callback = function(Value)
        if Value then
            local Players = game:GetService("Players")
            local RunService = game:GetService("RunService")
            local LocalPlayer = Players.LocalPlayer
            local AimbotConnection

            -- Role checks
            local function getRole()
                return LocalPlayer:FindFirstChild("Role") and LocalPlayer.Role.Value or nil
            end

            local function isSheriff()
                return getRole() == "Sheriff"
            end

            local function isMurderer()
                return getRole() == "Murderer"
            end

            -- Get the other role's player (exclude local player)
            local function getPlayerByRole(roleName)
                for _, player in ipairs(Players:GetPlayers()) do
                    if player ~= LocalPlayer and player:FindFirstChild("Role") and player.Role.Value == roleName then
                        return player
                    end
                end
                return nil
            end

            -- Smooth aim function
            local function smoothAimAt(targetPart)
                local camera = workspace.CurrentCamera
                if targetPart then
                    local currentCFrame = camera.CFrame
                    local targetCFrame = CFrame.new(camera.CFrame.Position, targetPart.Position)
                    camera.CFrame = currentCFrame:Lerp(targetCFrame, 0.2) -- 0.2 = smoothing speed
                end
            end

            -- Initial role check, automatically turn off if Innocent
            local role = getRole()
            if role ~= "Sheriff" and role ~= "Murderer" then
                Toggle:Set(false) -- Auto turn off if Innocent
                return
            end

            -- Aimbot loop
            AimbotConnection = RunService.RenderStepped:Connect(function()
                local currentRole = getRole()
                if currentRole ~= "Sheriff" and currentRole ~= "Murderer" then
                    -- Auto turn off if role changes to Innocent mid-game
                    Toggle:Set(false)
                    return
                end

                if currentRole == "Sheriff" then
                    local murderer = getPlayerByRole("Murderer")
                    if murderer and murderer.Character and murderer.Character:FindFirstChild("HumanoidRootPart") then
                        smoothAimAt(murderer.Character.HumanoidRootPart)
                    end
                elseif currentRole == "Murderer" then
                    local sheriff = getPlayerByRole("Sheriff")
                    if sheriff and sheriff.Character and sheriff.Character:FindFirstChild("HumanoidRootPart") then
                        smoothAimAt(sheriff.Character.HumanoidRootPart)
                    end
                end
            end)

            -- Save connection to disconnect later
            LocalPlayer:SetAttribute("SmoothAimbotConnection", AimbotConnection)

        else
            -- Turn off aimbot if toggle is off
            local LocalPlayer = game:GetService("Players").LocalPlayer
            local connection = LocalPlayer:GetAttribute("SmoothAimbotConnection")
            if connection then
                connection:Disconnect()
                LocalPlayer:SetAttribute("SmoothAimbotConnection", nil)
            end
        end
    end,
})
