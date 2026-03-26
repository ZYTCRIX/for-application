# for-application
To apply
-- Module und Services
local module3 = require(game.ReplicatedStorage:WaitForChild("Roundsystem"))

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local leaveEvent = game.ReplicatedStorage:WaitForChild("LeaveTarget")
local statusupdate = game.ReplicatedStorage:WaitForChild("UpdateTimer")
local toggleLeaveButton = game.ReplicatedStorage:WaitForChild("ToggleLeaveButton")

local TARGET_PLACE_ID = 130408978606735

local MAX_PLAYERS = 3

local targetToTeleport = {}

for teleport, target in pairs(module3) do
	targetToTeleport[target] = teleport
end

-- Spawn-Punkte
local basespawners = game.Workspace:WaitForChild("Map"):WaitForChild("BaseSpawner")
local randomspawners = {
	basespawners:WaitForChild("Spawn1"),
	basespawners:WaitForChild("Spawn2"),
	basespawners:WaitForChild("Spawn3"),
}


-- Countdown Länge
local intermissionLength = 10
local finishedIntermissions = 0

-- Pro Ziel-Part eigener Zustand
local partStates = {}

-- Spieler die gerade schon teleportiert werden
local teleporting = {}

-- Merkt sich in welcher Zone ein Spieler gerade ist
local playerCurrentTarget = {}


-- Initialisieren
for teleport, target in pairs(module3) do
	partStates[target] = {
		running = false,
		currentTime = intermissionLength,
		players = {},
	}
end


-- Spieler aus allen Zonen entfernen
local function removePlayerFromAllTargets(player)
	for _, state in pairs(partStates) do
		state.players[player] = nil
	end
end
--billboard update
local function updateBillboard(target)
	local teleport = targetToTeleport[target]
	if not teleport then return end

	local billboard = teleport:FindFirstChild("BillboardGui")
	if not billboard then return end

	local label = billboard:FindFirstChild("TextLabel")
	if not label then return end

	local state = partStates[target]
	if not state then return end

	local count = 0
	for _ in pairs(state.players) do
		count += 1
	end

	label.Text = count .. "/" .. MAX_PLAYERS .. " Players"
end

-- Countdown + Teleport
local function startIntermissionForPart(target)
	local state = partStates[target]
	if not state or state.running then return end

	state.running = true

	task.spawn(function()

		for i = intermissionLength, 0, -1 do
			if not state.running then return end
			state.currentTime = i
			
			for player in pairs(state.players) do
				if player and player.Parent == Players then
					statusupdate:FireClient(player, i)
				end
			end

			task.wait(1)
		end

		task.wait(2)

		local teleportList = {}

		for player in pairs(state.players) do
			if player
				and player.Parent == Players
				and not teleporting[player] then

				teleporting[player] = true
				table.insert(teleportList, player)
				statusupdate:FireClient(player, "Hide")
			end
		end

		if #teleportList > 0 then
			local success, err = pcall(function()
				TeleportService:TeleportPartyAsync(TARGET_PLACE_ID, teleportList)
			end)

			if not success then
				warn("Teleport failed:", err)

				-- Schutz: falls es fehlschlägt, Flag wieder freigeben
				for _, plr in ipairs(teleportList) do
					teleporting[plr] = nil
				end
			end
		end

		task.wait(2)

		state.running = false
		state.currentTime = intermissionLength
		state.players = {}
		updateBillboard(target)
		finishedIntermissions += 1

	end)
end

leaveEvent.OnServerEvent:Connect(function(player)

	local target = playerCurrentTarget[player]
	if not target then return end

	local playerGui = player:WaitForChild("PlayerGui")
	if not playerGui then return end
	
	local state = partStates[target]
	if not state then return end
	
	local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
	if not hrp then return end
	-- Spieler entfernen
	hrp.CFrame = workspace:WaitForChild("Respawn").CFrame
	state.players[player] = nil
	playerCurrentTarget[player] = nil

	updateBillboard(target)
	playerGui:WaitForChild("SideGui").Enabled = true
	toggleLeaveButton:FireClient(player, false)
	if next(state.players) == nil then
		state.running = false
		state.currentTime = intermissionLength
		statusupdate:FireClient(player, "Hide")
	end
end)

-- Lobby Pads und Countdown Zonen
for teleport, target in pairs(module3) do

	teleport.Touched:Connect(function(hit)
		local character = hit.Parent
		if not character then return end

		local player = Players:GetPlayerFromCharacter(character)
		if not player then return end

		local hrp = character:FindFirstChild("HumanoidRootPart")
		if not hrp then return end
		
		hrp.CFrame = target.CFrame + Vector3.new(0, 5, 0)
	end)


	-- Countdown Zone betreten
	target.Touched:Connect(function(hit)

		if hit.Name ~= "HumanoidRootPart" then return end

		local character = hit.Parent
		if not character then return end

		local player = Players:GetPlayerFromCharacter(character)
		local playerGui = player:WaitForChild("PlayerGui")
		playerGui:WaitForChild("SideGui").Enabled = false
		if not player then return end
		if teleporting[player] then return end

		local state = partStates[target]
		if not state then return end

		if playerCurrentTarget[player] == target then
			return
		end

		if playerCurrentTarget[player] and playerCurrentTarget[player] ~= target then
			removePlayerFromAllTargets(player)
		end

		playerCurrentTarget[player] = target
		state.players[player] = true
		toggleLeaveButton:FireClient(player, true)
		updateBillboard(target)
		

		if state.running then
			statusupdate:FireClient(player, state.currentTime)
		else
			startIntermissionForPart(target)
		end
	end)
	


	-- Countdown Zone verlassen
	target.TouchEnded:Connect(function(hit)

		if hit.Name ~= "HumanoidRootPart" then return end

		local character = hit.Parent
		if not character then return end

		local player = Players:GetPlayerFromCharacter(character)
		local playerGui = player:WaitForChild("PlayerGui")
		if not player then return end

		if playerCurrentTarget[player] ~= target then
			return
		end

		local state = partStates[target]
		if not state then return end
		
		playerGui:WaitForChild("SideGui").Enabled = true
		state.players[player] = nil
		playerCurrentTarget[player] = nil
		toggleLeaveButton:FireClient(player, false)
		updateBillboard(target)
		
		if next(state.players) == nil then
			state.running = false
			state.currentTime = intermissionLength
			statusupdate:FireClient(player, "Hide")
		end
	end)
end
-- Cleanup
Players.PlayerRemoving:Connect(function(player)
	teleporting[player] = nil
	playerCurrentTarget[player] = nil
	removePlayerFromAllTargets(player)
	for target in pairs(partStates) do
		updateBillboard(target)
	end
end)
