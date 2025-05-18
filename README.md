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

        local function updateESPColors()
            for _, player in pairs(Players:GetPlayers()) do
                if player ~= LocalPlayer and player.Character then
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
            -- Apply to existing players
            for _, player in pairs(Players:GetPlayers()) do
                if player ~= LocalPlayer then
                    createESP(player)
                end
            end

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
