task.spawn(function()
        local load = require(game.ReplicatedStorage:WaitForChild("Fsys")).load

        set_thread_identity(2)
        local clientData = load("ClientData")
        local items = load("KindDB")
        local router = load("RouterClient")
        local downloader = load("DownloadClient")
        local animationManager = load("AnimationManager")
        local petRigs = load("new:PetRigs")
        set_thread_identity(8)
        
        
        local petModels = {}
        local pets = {}
        local equippedPet = nil
        local mountedPet = nil
        local currentMountTrack = nil
        
        
        local function updateData(key, action)
            local data = clientData.get(key)
        
            local clonedData = table.clone(data)
            clientData.predict(key, action(clonedData))
        end
        
        local function getUniqueId()
            local HttpService = game:GetService("HttpService")
            return HttpService:GenerateGUID(false)
        end
        
        local function getPetModel(kind)
            if petModels[kind] then
                return petModels[kind]
            end
        
            local streamed = downloader.promise_download_copy("Pets", kind):expect()
            petModels[kind] = streamed
            return streamed
        end
        
        local function createPet(id, properties)
            local uniqueId = getUniqueId()
            local pet = nil
        
            set_thread_identity(2)
        
            updateData("inventory", function(inventory)
                local newPets = table.clone(inventory.pets)
                local item = items[id]
        
                pet = {
                    unique = uniqueId,
                    category = "pets",
                    id = id,
                    kind = item.kind,
                    newness_order = 0,
                    properties = properties
                }
        
                newPets[uniqueId] = pet
                inventory.pets = newPets
        
                return inventory
            end)
        
            set_thread_identity(8)
        
            pets[uniqueId] = {
                data = pet,
                model = nil
            }
        
            return pet
        end
        
        local function neonify(model, entry)
            local petModel = model:FindFirstChild("PetModel")
        
            if not petModel then
                return
            end
        
            for neonPart, configuration in pairs(entry.neon_parts) do
                local trueNeonPart = petRigs.get(petModel).get_geo_part(petModel, neonPart)
                trueNeonPart.Material = configuration.Material
                trueNeonPart.Color = configuration.Color
            end
        end
        
        local function addPetWrapper(wrapper)
            updateData("pet_char_wrappers", function(petWrappers)
                wrapper.unique = #petWrappers + 1
                wrapper.index = #petWrappers + 1
                petWrappers[#petWrappers + 1] = wrapper
                return petWrappers
            end)
        end
        
        local function addPetState(state)
            updateData("pet_state_managers", function(petStates)
                petStates[#petStates + 1] = state
                return petStates
            end)
        end
        
        local function findIndex(array, finder)
            for index, value in pairs(array) do
                local isIt = finder(value, index)
        
                if isIt then
                    return index
                end
            end
        
            return nil
        end
        
        local function removePetWrapper(uniqueId)
            updateData("pet_char_wrappers", function(petWrappers)
                local index = findIndex(petWrappers, function(wrapper)
                    return wrapper.pet_unique == uniqueId
                end)
        
                if not index then
                    return petWrappers
                end
        
                table.remove(petWrappers, index)
        
                for wrapperIndex, wrapper in pairs(petWrappers) do
                    wrapper.unique = wrapperIndex
                    wrapper.index = wrapperIndex
                end
        
                return petWrappers
            end)
        end
        
        local function clearPetState(uniqueId)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            updateData("pet_state_managers", function(states)
                local index = findIndex(states, function(state)
                    return state.char == pet.model
                end)
        
                if not index then
                    return states
                end
        
                local clonedStates = table.clone(states)
        
                clonedStates[index] = table.clone(clonedStates[index])
                clonedStates[index].states = {}
        
                return clonedStates
            end)
        end
        
        
        local function setPetState(uniqueId, id)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            updateData("pet_state_managers", function(states)
                local index = findIndex(states, function(state)
                    return state.char == pet.model
                end)
        
                if not index then
                    return states
                end
        
                local clonedStates = table.clone(states)
        
                clonedStates[index] = table.clone(clonedStates[index])
                clonedStates[index].states = {
                    { id = id }
                }
        
                return clonedStates
            end)
        end
        
        local function attachPlayerToPet(pet)
            local character = game.Players.LocalPlayer.Character
        
            if not character then
                return false
            end
        
            if not character.PrimaryPart then
                return false
            end
        
            local ridePosition = pet:FindFirstChild("RidePosition", true)
        
            if not ridePosition then
                return false
            end
        
            local sourceAttachment = Instance.new("Attachment")
        
            sourceAttachment.Parent = ridePosition
            sourceAttachment.Position = Vector3.new(0, 1.237, 0)
            sourceAttachment.Name = "SourceAttachment"
        
            local stateConnection = Instance.new("RigidConstraint")
        
            stateConnection.Name = "StateConnection"
            stateConnection.Attachment0 = sourceAttachment
            stateConnection.Attachment1 = character.PrimaryPart.RootAttachment
        
            stateConnection.Parent = character
        
            return true
        end
        
        
        local function clearPlayerState()
            updateData("state_manager", function(state)
                local clonedState = table.clone(state)
                clonedState.states = {}
                clonedState.is_sitting = false
                return clonedState
            end)
        end
        
        
        local function setPlayerState(id)
            updateData("state_manager", function(state)
                local clonedState = table.clone(state)
        
                clonedState.states = {
                    { id = id }
                }
        
                clonedState.is_sitting = true
        
                return clonedState
            end)
        end
        
        
        local function removePetState(uniqueId)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            updateData("pet_state_managers", function(petStates)
                local index = findIndex(petStates, function(state)
                    return state.char == pet.model
                end)
        
                if not index then
                    return petStates
                end
        
                table.remove(petStates, index)
                return petStates
            end)
        end
        
        local function unmount(uniqueId)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            if currentMountTrack then
                currentMountTrack:Stop()
                currentMountTrack:Destroy()
            end
        
            local sourceAttachment = pet.model:FindFirstChild("SourceAttachment", true)
        
            if sourceAttachment then
                sourceAttachment:Destroy()
            end
        
            if game.Players.LocalPlayer.Character then
                for _, descendant in pairs(game.Players.LocalPlayer.Character:GetDescendants()) do
                    if descendant:IsA("BasePart") and descendant:GetAttribute("HaveMass") then
                        descendant.Massless = false
                    end
                end
            end
        
            clearPetState(uniqueId)
            clearPlayerState()
        
            pet.model:ScaleTo(1)
        
            mountedPet = nil
        end
        
        local function mount(uniqueId, playerState, petState)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            local player = game.Players.LocalPlayer
        
            if not player.Character then
                return
            end
        
            if not player.Character.PrimaryPart then
                return
            end
        
            mountedPet = uniqueId
        
            setPetState(uniqueId, petState)
            setPlayerState(playerState)
        
            pet.model:ScaleTo(2)
            attachPlayerToPet(pet.model)
        
            currentMountTrack = player.Character.Humanoid.Animator:LoadAnimation(animationManager.get_track("PlayerRidingPet"))
            player.Character.Humanoid.Sit = true
        
            for _, descendant in pairs(player.Character:GetDescendants()) do
                if descendant:IsA("BasePart") and descendant.Massless == false then
                    descendant.Massless = true
                    descendant:SetAttribute("HaveMass", true)
                end
            end
        
            currentMountTrack:Play()
        end
        
        local function fly(uniqueId)
            mount(uniqueId, "PlayerFlyingPet", "PetBeingFlown")
        end
        
        local function ride(uniqueId)
            mount(uniqueId, "PlayerRidingPet", "PetBeingRidden")
        end
        
        local function unequip(item)
            local pet = pets[item.unique]
        
            if not pet then
                return
            end
        
            if not pet.model then
                return
            end
        
            unmount(item.unique)
        
            removePetWrapper(item.unique)
            removePetState(item.unique)
        
            pet.model:Destroy()
            pet.model = nil
        
            equippedPet = nil
        end
        
        local function equip(item)
            if equippedPet then
                unequip(equippedPet)
            end
        
            local petModel = getPetModel(item.kind):Clone()
        
            petModel.Parent = workspace
            pets[item.unique].model = petModel
        
            if item.properties.neon or item.properties.mega_neon then
                neonify(petModel, items[item.kind])
            end
        
            equippedPet = item
        
            addPetWrapper({
                char = petModel,
                mega_neon = item.properties.mega_neon,
                neon = item.properties.neon,
                player = game.Players.LocalPlayer,
                entity_controller = game.Players.LocalPlayer,
                controller = game.Players.LocalPlayer,
                rp_name = item.properties.rp_name or "",
                pet_trick_level = item.properties.pet_trick_level,
                pet_unique = item.unique,
                pet_id = item.id,
                location = {
                    full_destination_id = "housing",
                    destination_id = "housing",
                    house_owner = game.Players.LocalPlayer
                },
                pet_progression = {
                    friendship_level = item.properties.friendship_level,
                    age = item.properties.age,
                    percentage = 0
                },
                are_colors_sealed = false,
                is_pet = true
            })
        
            addPetState({
                char = petModel,
                player = game.Players.LocalPlayer,
                store_key = "pet_state_managers",
                is_sitting = false,
                chars_connected_to_me = {},
                states = {}
            })
        end
        
        local oldGet = router.get
        
        local function createRemoteFunctionMock(callback)
            return {
                InvokeServer = function(_, ...)
                    return callback(...)
                end
            }
        end
        
        local function createRemoteEventMock(callback)
            return {
                FireServer = function(_, ...)
                    return callback(...)
                end
            }
        end
        
        local equipRemote = createRemoteFunctionMock(function(uniqueId, metadata)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            equip(pet.data)
        
            return true, {
                action = "equip",
                is_server = true
            }
        end)
        
        local unequipRemote = createRemoteFunctionMock(function(uniqueId)
            local pet = pets[uniqueId]
        
            if not pet then
                return
            end
        
            unequip(pet.data)
        
            return true, {
                action = "unequip",
                is_server = true
            }
        end)
        
        local rideRemote = createRemoteFunctionMock(function(item)
            ride(item.pet_unique)
        end)
        
        local flyRemote = createRemoteFunctionMock(function(item)
            fly(item.pet_unique)
        end)
        
        local unmountRemoteFunction = createRemoteFunctionMock(function()
            unmount(mountedPet)
        end)
        
        local unmountRemoteEvent = createRemoteEventMock(function()
            unmount(mountedPet)
        end)
        
        router.get = function(name)
            if name == "ToolAPI/Equip" then
                return equipRemote
            end
        
            if name == "ToolAPI/Unequip" then
                return unequipRemote
            end
        
            if name == "AdoptAPI/RidePet" then
                return rideRemote
            end
        
            if name == "AdoptAPI/FlyPet" then
                return flyRemote
            end
        
            if name == "AdoptAPI/ExitSeatStatesYield" then
                return unmountRemoteFunction
            end
        
            if name == "AdoptAPI/ExitSeatStates" then
                return unmountRemoteEvent
            end
        
            return oldGet(name)
        end
        
        for _, charWrapper in pairs(clientData.get("pet_char_wrappers")) do
            oldGet("ToolAPI/Unequip"):InvokeServer(charWrapper.pet_unique)
        end
        
        local Loads = require(game.ReplicatedStorage.Fsys).load
        local InventoryDB = Loads("InventoryDB")
        
        function GetPetByName(name)
            for i,v in pairs(InventoryDB.pets) do
                if v.name:lower() == name:lower() then
                    return v.id
                end
            end
            return false
        end
        
        local Players = game:GetService("Players")
        local LocalPlayer = Players.LocalPlayer
        local UserInputService = game:GetService("UserInputService")
        
        -- UI Setup
        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
        
        local Frame = Instance.new("Frame")
        Frame.Size = UDim2.new(0, 350, 0, 250)
        Frame.Position = UDim2.new(0.5, -175, 0.5, -125)
        Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
        Frame.BackgroundTransparency = 0.1
        Frame.BorderSizePixel = 0
        Frame.Active = true
        Frame.Draggable = true
        Frame.Parent = ScreenGui
        
        -- Rounded Corners
        local UICorner = Instance.new("UICorner")
        UICorner.CornerRadius = UDim.new(0, 10)
        UICorner.Parent = Frame
        
        -- Rainbow Border Effect
        local UIStroke = Instance.new("UIStroke")
        UIStroke.Thickness = 3  
        UIStroke.Parent = Frame
        
        local function animateRainbow()
            while true do
                for hue = 0, 1, 0.01 do
                    UIStroke.Color = Color3.fromHSV(hue, 1, 1)  
                    task.wait(0.05)  
                end
            end
        end
        
        task.spawn(animateRainbow)
        
        -- Title Bar
        local TitleBar = Instance.new("Frame")
        TitleBar.Size = UDim2.new(1, 0, 0, 30)
        TitleBar.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
        TitleBar.BorderSizePixel = 0
        TitleBar.Parent = Frame
        
        local TitleText = Instance.new("TextLabel")
        TitleText.Text = "Pet Spawner"
        TitleText.Size = UDim2.new(1, 0, 1, 0)
        TitleText.BackgroundTransparency = 1
        TitleText.TextColor3 = Color3.fromRGB(255, 255, 255)
        TitleText.Font = Enum.Font.GothamBold
        TitleText.TextSize = 16
        TitleText.Parent = TitleBar
        
        -- Input Box
        local TextBox = Instance.new("TextBox")
        TextBox.Size = UDim2.new(0.8, 0, 0, 40)
        TextBox.Position = UDim2.new(0.1, 0, 0.25, 0)
        TextBox.PlaceholderText = "Enter Pet Name"
        TextBox.Text = ""
        TextBox.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
        TextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
        TextBox.Font = Enum.Font.Gotham
        TextBox.TextSize = 14
        TextBox.Parent = Frame
        
        local UICorner2 = Instance.new("UICorner")
        UICorner2.CornerRadius = UDim.new(0, 8)
        UICorner2.Parent = TextBox
        
        -- Pet Type Selection
        local petType = "FR"  
        -- Feedback Text
        local Feedback = Instance.new("TextLabel")
        Feedback.Size = UDim2.new(1, 0, 0, 30)
        Feedback.Position = UDim2.new(0, 0, 0.85, 0)
        Feedback.BackgroundTransparency = 1
        Feedback.Text = ""
        Feedback.TextColor3 = Color3.fromRGB(255, 255, 255)
        Feedback.Font = Enum.Font.Gotham
        Feedback.TextSize = 14
        Feedback.Parent = Frame
        
        local function createTypeButton(name, position, color)
            local button = Instance.new("TextButton")
            button.Size = UDim2.new(0.25, 0, 0, 30)
            button.Position = UDim2.new(position, 0, 0.45, 0)
            button.Text = name
            button.BackgroundColor3 = color
            button.TextColor3 = Color3.fromRGB(255, 255, 255)
            button.Font = Enum.Font.GothamBold
            button.TextSize = 14
            button.Parent = Frame
        
            local UICorner = Instance.new("UICorner")
            UICorner.CornerRadius = UDim.new(0, 8)
            UICorner.Parent = button
        
            button.MouseButton1Click:Connect(function()
                petType = name
                Feedback.Text = "Selected: " .. name
                Feedback.TextColor3 = Color3.fromRGB(255, 255, 0)
            end)
        
            return button
        end
        
        local MFRButton = createTypeButton("MFR", 0.1, Color3.fromRGB(255, 100, 100))
        local NFRButton = createTypeButton("NFR", 0.4, Color3.fromRGB(100, 255, 100))
        local FRButton = createTypeButton("FR", 0.7, Color3.fromRGB(100, 100, 255))
        -- Duplicate Button
        local Button = Instance.new("TextButton")
        Button.Size = UDim2.new(0.8, 0, 0, 40)
        Button.Position = UDim2.new(0.1, 0, 0.65, 0)
        Button.Text = "Spawn Pet"
        Button.BackgroundColor3 = Color3.fromRGB(60, 120, 200)
        Button.TextColor3 = Color3.fromRGB(255, 255, 255)
        Button.Font = Enum.Font.GothamBold
        Button.TextSize = 14
        Button.Parent = Frame
        
        local UICorner3 = Instance.new("UICorner")
        UICorner3.CornerRadius = UDim.new(0, 8)
        UICorner3.Parent = Button
        
        Button.MouseButton1Click:Connect(function()
            local petName = GetPetByName(TextBox.Text)
            if petName then
                if petType == 'FR' then
                    createPet(petName, {
                        pet_trick_level = 0,
                        rideable = true,
                        flyable = true,
                        friendship_level = 0,
                        age = 1,
                        ailments_completed = 0,
                        rp_name = ""
                    })
                elseif petType == 'MFR' then
                    createPet(petName, {
                        pet_trick_level = 0,
                        mega_neon = true,
                        rideable = true,
                        flyable = true,
                        friendship_level = 0,
                        age = 1,
                        ailments_completed = 0,
                        rp_name = ""
                    })
                elseif petType == 'NFR' then
                    createPet(petName, {
                        pet_trick_level = 0,
                        neon = true,
                        rideable = true,
                        flyable = true,
                        friendship_level = 0,
                        age = 1,
                        ailments_completed = 0,
                        rp_name = ""
                    })
                end
            end
        end)
    end)
