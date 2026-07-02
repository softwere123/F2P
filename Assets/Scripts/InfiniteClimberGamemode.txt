-- Infinite Platform Climber gamemode for Gorilla Tag custom maps.
-- Implements: STEP1 scoring platforms w/ pooling+recycle, STEP2 rising death zone + reset,
-- STEP3 in-run buff gacha at score thresholds, STEP4 post-death cosmetic-reward gacha.
--
-- All touch detection is proximity-based (distance checks against LocalPlayer.leftHandPosition/
-- rightHandPosition/bodyPosition each tick) rather than LuauTrigger/:onTouched() callbacks. This
-- means every required scene object is a plain GameObject - no LuauTrigger component needed
-- anywhere, so the whole setup can be built without any Editor-only component wiring.
--
-- Required objects in the initial scene (see setup notes at the bottom of this file):
--   CustomMapSpawnPoint     - respawn target (the standard CustomMapSpawnPoint prefab works)
--   Platform_1..Platform_N  - solid (non-trigger) platforms, not Static
--   DeathZone                - a plain GameObject; death fires once the player's Y drops to/below it
--   CosmeticReward_1..3      - template objects to clone as gacha rewards, not Static
--   RewardPedestal            - plain GameObject, where granted rewards are placed

-- ================= CONFIG =================
local VERTICAL_SPACING = 2.5
local RECYCLE_MARGIN_BELOW_PLAYER = 4.0
local HORIZONTAL_RANGE = 2.0
local PLATFORM_TOUCH_RADIUS = 1.0

local DEATHZONE_RISE_SPEED = 1.0
local DEATHZONE_OFFSET_BELOW_PLAYER = 5.0

local GACHA_THRESHOLDS = { [2] = true, [4] = true, [8] = true }
local JUMP_BUFF_DURATION = 8.0
local JUMP_BUFF_MULTIPLIER_FACTOR = 2.0
local PLATFORM_PULL_AMOUNT = 1.5

local REWARD_SPACING = 1.0

-- ================= TIMER UTILITY (retry-based init, same pattern as the SDK examples) =================
local DelayedFunctions = {}

function setTimeout(delay, func)
	table.insert(DelayedFunctions, { endTime = os.clock() + delay, callback = func })
end

function handleTimeouts()
	local now = os.clock()
	for i = #DelayedFunctions, 1, -1 do
		local t = DelayedFunctions[i]
		if now >= t.endTime then
			t.callback()
			table.remove(DelayedFunctions, i)
		end
	end
end

-- ================= STATE =================
local scriptInitialized = false

local score = 0
local isGameOver = false
local usedThresholds = {}

local baseJumpMultiplier = nil
local jumpBuffEndTime = nil

local rollsAvailable = 0
local rewardsGranted = 0

local platforms = {} -- array of { obj = GameObject, touched = bool }
local spawnPoint = nil
local deathZone = nil
local cosmeticTemplates = {}
local rewardPedestal = nil

-- ================= PLATFORMS (STEP 1) =================
function findPlatforms()
	local i = 1
	while true do
		local p = GameObject.findGameObject("Platform_" .. tostring(i))
		if p == nil then break end
		table.insert(platforms, { obj = p, touched = false })
		i += 1
	end
end

function highestPlatformY()
	local highest = spawnPoint.position.y
	for _, entry in ipairs(platforms) do
		if entry.obj.position.y > highest then
			highest = entry.obj.position.y
		end
	end
	return highest
end

function randomizedPosition(y)
	local ox = spawnPoint.position.x + (math.random() * 2 - 1) * HORIZONTAL_RANGE
	local oz = spawnPoint.position.z + (math.random() * 2 - 1) * HORIZONTAL_RANGE
	return Vec3.new(ox, y, oz)
end

function armPlatform(entry)
	entry.touched = false
	entry.obj:setColor(Vec3.new(1, 1, 1))
end

function arrangeInitialStack()
	local y = spawnPoint.position.y
	for _, entry in ipairs(platforms) do
		y = y + VERTICAL_SPACING
		entry.obj.position = randomizedPosition(y)
		armPlatform(entry)
	end
end

function onPlatformTouched(entry)
	if entry.touched or isGameOver then return end
	entry.touched = true
	entry.obj:setColor(Vec3.new(0, 1, 0))
	addScore(1)
end

-- Hand/body proximity check against one platform, in place of a trigger callback.
function isPlayerTouchingPlatform(entry)
	local pos = entry.obj.position
	if Vec3.distance(pos, LocalPlayer.bodyPosition) <= PLATFORM_TOUCH_RADIUS then return true end
	if Vec3.distance(pos, LocalPlayer.leftHandPosition) <= PLATFORM_TOUCH_RADIUS then return true end
	if Vec3.distance(pos, LocalPlayer.rightHandPosition) <= PLATFORM_TOUCH_RADIUS then return true end
	return false
end

-- Each tick: score untouched platforms the player is near, and recycle any platform that has
-- fallen too far below the player back to the top of the stack.
function updatePlatforms()
	local playerY = LocalPlayer.bodyPosition.y
	for _, entry in ipairs(platforms) do
		if not entry.touched and isPlayerTouchingPlatform(entry) then
			onPlatformTouched(entry)
		end

		if entry.obj.position.y < playerY - RECYCLE_MARGIN_BELOW_PLAYER then
			local newY = highestPlatformY() + VERTICAL_SPACING
			entry.obj.position = randomizedPosition(newY)
			armPlatform(entry)
		end
	end
end

-- Used by the STEP3 "pull the next platform closer" buff.
function getNextPlatformAbovePlayer()
	local playerY = LocalPlayer.bodyPosition.y
	local nearest = nil
	local nearestY = math.huge
	for _, entry in ipairs(platforms) do
		local y = entry.obj.position.y
		if y > playerY and y < nearestY then
			nearest = entry
			nearestY = y
		end
	end
	return nearest
end

-- ================= SCORE / GACHA THRESHOLDS =================
function addScore(amount)
	if isGameOver then return end
	score = score + amount
	print("Score: " .. tostring(score))
	checkGachaThreshold()
end

function checkGachaThreshold()
	if GACHA_THRESHOLDS[score] and not usedThresholds[score] then
		usedThresholds[score] = true
		rollBuff()
	end
end

-- ================= STEP 3: IN-RUN BUFF GACHA =================
function rollBuff()
	if math.random(0, 1) == 0 then
		applyJumpBuff()
	else
		pullNextPlatformCloser()
	end
end

function applyJumpBuff()
	if baseJumpMultiplier == nil then
		baseJumpMultiplier = PlayerSettings.jumpMultiplier
	end
	PlayerSettings.jumpMultiplier = baseJumpMultiplier * JUMP_BUFF_MULTIPLIER_FACTOR
	jumpBuffEndTime = os.clock() + JUMP_BUFF_DURATION
	print("Buff rolled: jump x" .. tostring(JUMP_BUFF_MULTIPLIER_FACTOR) .. " for " .. tostring(JUMP_BUFF_DURATION) .. "s")
end

function updateJumpBuff()
	if jumpBuffEndTime ~= nil and os.clock() >= jumpBuffEndTime then
		PlayerSettings.jumpMultiplier = baseJumpMultiplier
		jumpBuffEndTime = nil
	end
end

function pullNextPlatformCloser()
	local nextPlatform = getNextPlatformAbovePlayer()
	if nextPlatform == nil then return end

	local minY = LocalPlayer.bodyPosition.y + 1.0
	local pos = nextPlatform.obj.position
	local newY = math.max(minY, pos.y - PLATFORM_PULL_AMOUNT)
	nextPlatform.obj.position = Vec3.new(pos.x, newY, pos.z)
	print("Buff rolled: next platform pulled closer")
end

-- ================= STEP 2: DEATH ZONE + RESET =================
function updateDeathZone(dt)
	if deathZone == nil then return end

	local targetY = LocalPlayer.bodyPosition.y - DEATHZONE_OFFSET_BELOW_PLAYER
	if targetY > deathZone.position.y then -- ratchet: never move back down
		local pos = deathZone.position
		local newY = math.min(targetY, pos.y + DEATHZONE_RISE_SPEED * dt)
		deathZone.position = Vec3.new(pos.x, newY, pos.z)
	end

	if LocalPlayer.bodyPosition.y <= deathZone.position.y then
		handlePlayerDeath()
	end
end

function handlePlayerDeath()
	if isGameOver then return end
	isGameOver = true

	grantCosmeticRolls(score)
	resetGame()
end

function resetGame()
	score = 0
	isGameOver = false
	usedThresholds = {}
	baseJumpMultiplier = nil
	jumpBuffEndTime = nil

	respawnPlayer()
	arrangeInitialStack()
end

function respawnPlayer()
	if spawnPoint == nil then return end
	setPlayerPosition(spawnPoint.position, false) -- false = don't keep velocity
end

-- ================= STEP 4: POST-DEATH COSMETIC GACHA =================
function findCosmetics()
	local i = 1
	while true do
		local c = GameObject.findGameObject("CosmeticReward_" .. tostring(i))
		if c == nil then break end
		table.insert(cosmeticTemplates, c)
		i += 1
	end
end

-- 1 score point = 1 gacha roll, granted immediately on death.
function grantCosmeticRolls(scoreThisRun)
	if scoreThisRun <= 0 then return end
	rollsAvailable = rollsAvailable + scoreThisRun
	print("Cosmetic gacha rolls granted: " .. tostring(scoreThisRun) .. " (total: " .. tostring(rollsAvailable) .. ")")
	rollAllCosmetics()
end

function rollAllCosmetics()
	while rollsAvailable > 0 and #cosmeticTemplates > 0 do
		rollOneCosmetic()
	end
end

function rollOneCosmetic()
	if rollsAvailable <= 0 or #cosmeticTemplates == 0 then return end
	rollsAvailable = rollsAvailable - 1

	local template = cosmeticTemplates[math.random(1, #cosmeticTemplates)]
	local reward = template:clone()
	reward:setVisibility(true)
	reward:setActive(true)

	local basePos = spawnPoint ~= nil and spawnPoint.position or template.position
	if rewardPedestal ~= nil then
		basePos = rewardPedestal.position
	end
	reward.position = basePos + Vec3.new(rewardsGranted * REWARD_SPACING, 0, 0)
	rewardsGranted += 1

	print("Cosmetic reward granted (" .. tostring(rollsAvailable) .. " rolls left)")
end

-- ================= INIT / TICK =================
function initGame()
	if LocalPlayer == nil or InRoom == false then
		setTimeout(1, initGame)
		return
	end

	spawnPoint = GameObject.findGameObject("CustomMapSpawnPoint")
	if spawnPoint == nil then
		print("[initGame] CustomMapSpawnPoint not found yet, retrying...")
		setTimeout(1, initGame)
		return
	end

	findPlatforms()
	if #platforms == 0 then
		print("[initGame] no Platform_N objects found yet, retrying...")
		setTimeout(1, initGame)
		return
	end

	deathZone = GameObject.findGameObject("DeathZone")
	rewardPedestal = GameObject.findGameObject("RewardPedestal")
	findCosmetics()

	if deathZone == nil then
		print("[initGame] WARNING: DeathZone not found, falling off will not end the run")
	end

	arrangeInitialStack()

	scriptInitialized = true
	print("[initGame] Infinite climber gamemode ready with " .. tostring(#platforms) .. " platforms.")
end

function tick(dt)
	handleTimeouts()

	if not scriptInitialized then return end
	if isGameOver then return end

	updatePlatforms()
	updateDeathZone(dt)
	updateJumpBuff()
end

initGame()

--[[
	SETUP CHECKLIST (do this in the Unity Editor, in your initial scene):

	All of the objects below already exist in Assets/Scenes/MyFirstScnce.unity - this list is
	just for reference / for building a different scene from scratch.

	1. CustomMapSpawnPoint  - the standard CustomMapSpawnPoint prefab instance.
	2. Platform_1 .. Platform_N (named exactly this way, sequential, no gaps)
	                          - solid (non-trigger) Collider, NOT Static (Lua repositions them).
	3. DeathZone              - plain GameObject, NOT Static, placed below your platforms.
	                            No collider needed - detection is a Y-position check.
	4. CosmeticReward_1, CosmeticReward_2, CosmeticReward_3
	                          - your cosmetic reward objects, NOT Static (Lua clones+moves them).
	5. RewardPedestal (optional) - empty GameObject marking where granted rewards line up.

	6. This file is Assets/Scripts/InfiniteClimberGamemode.txt - already wired into the initial
	   scene's Map Descriptor "Custom Gamemode" field.
	7. To test in-game: select CUSTOM in the game mode selector inside the Virtual Stump,
	   then load the map.
--]]
