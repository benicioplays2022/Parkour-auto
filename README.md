-- SCRIPT PARKOUR COMPLETO - BENÍCIO - JULHO 2025

local player = game.Players.LocalPlayer
local mouse = player:GetMouse()
local char = player.Character or player.CharacterAdded:Wait()
local hum = char:WaitForChild("Humanoid")
local hrp = char:WaitForChild("HumanoidRootPart")
local PathfindingService = game:GetService("PathfindingService")
local UserInputService = game:GetService("UserInputService")

local points = {}
local routes = {}
local editMode = false
local routeName = "default"
local pointType = "Normal"
local visualMode = "Button"

local visuals = {}

local function clearVisuals()
	for _, v in pairs(visuals) do
		if v and v.Parent then
			v:Destroy()
		end
	end
	visuals = {}
end

-- GUI --------------------------------------

local gui = Instance.new("ScreenGui")
gui.Name = "ParkourUI"
gui.ResetOnSpawn = false
gui.Parent = player:WaitForChild("PlayerGui")

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 300, 0, 420)
frame.Position = UDim2.new(0, 10, 0.5, -210)
frame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
frame.BorderSizePixel = 0
frame.Active = true
frame.Parent = gui

-- DRAG -------------------------------------

local dragging, dragInput, dragStart, startPos = false, nil, nil, nil

local function update(input)
	local delta = input.Position - dragStart
	frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
							 startPos.Y.Scale, startPos.Y.Offset + delta.Y)
end

frame.InputBegan:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 then
		dragging = true
		dragStart = input.Position
		startPos = frame.Position
		input.Changed:Connect(function()
			if input.UserInputState == Enum.UserInputState.End then dragging = false end
		end)
	end
end)

frame.InputChanged:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseMovement then
		dragInput = input
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if input == dragInput and dragging then
		update(input)
	end
end)

-- MINIMIZAR/MAXIMIZAR ----------------------

local btnMinimize = Instance.new("TextButton", frame)
btnMinimize.Size = UDim2.new(0, 30, 0, 30)
btnMinimize.Position = UDim2.new(1, -60, 0, 5)
btnMinimize.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
btnMinimize.TextColor3 = Color3.fromRGB(255, 255, 255)
btnMinimize.Font = Enum.Font.GothamBold
btnMinimize.TextSize = 24
btnMinimize.Text = "—"

local btnMaximize = Instance.new("TextButton", frame)
btnMaximize.Size = UDim2.new(0, 30, 0, 30)
btnMaximize.Position = UDim2.new(1, -30, 0, 5)
btnMaximize.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
btnMaximize.TextColor3 = Color3.fromRGB(255, 255, 255)
btnMaximize.Font = Enum.Font.GothamBold
btnMaximize.TextSize = 20
btnMaximize.Text = "⬜"

local isMinimized = false
local defaultSize = frame.Size
local defaultPosition = frame.Position

local tabButtons = {}
local tabContents = {}

btnMinimize.MouseButton1Click:Connect(function()
	if not isMinimized then
		for _, content in pairs(tabContents) do content.Visible = false end
		for _, btn in pairs(tabButtons) do btn.Visible = false end
		isMinimized = true
		frame.Size = UDim2.new(frame.Size.X.Scale, frame.Size.X.Offset, 0, 40)
	else
		tabButtons["Editor"].Visible = true
		tabContents["Editor"].Visible = true
		isMinimized = false
		frame.Size = defaultSize
	end
end)

btnMaximize.MouseButton1Click:Connect(function()
	if frame.Size == defaultSize then
		frame.Size = UDim2.new(0, 600, 0, 700)
		frame.Position = UDim2.new(0.5, -300, 0.5, -350)
	else
		frame.Size = defaultSize
		frame.Position = defaultPosition
	end
end)

-- TABS ------------------------------------

local tabsFrame = Instance.new("Frame", frame)
tabsFrame.Size = UDim2.new(1, 0, 0, 30)
tabsFrame.Position = UDim2.new(0, 0, 0, 40)
tabsFrame.BackgroundTransparency = 1

local function createTab(name, posX)
	local btn = Instance.new("TextButton", tabsFrame)
	btn.Size = UDim2.new(0, 90, 1, 0)
	btn.Position = UDim2.new(0, posX, 0, 0)
	btn.Text = name
	btn.Font = Enum.Font.GothamBold
	btn.TextSize = 16
	btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	btn.TextColor3 = Color3.fromRGB(180, 180, 180)

	local content = Instance.new("Frame", frame)
	content.Size = UDim2.new(1, -20, 1, -80)
	content.Position = UDim2.new(0, 10, 0, 70)
	content.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
	content.Visible = false

	tabButtons[name] = btn
	tabContents[name] = content

	btn.MouseButton1Click:Connect(function()
		for k,v in pairs(tabContents) do
			v.Visible = false
			tabButtons[k].BackgroundColor3 = Color3.fromRGB(40, 40, 40)
			tabButtons[k].TextColor3 = Color3.fromRGB(180, 180, 180)
		end
		content.Visible = true
		btn.BackgroundColor3 = Color3.fromRGB(0, 200, 120)
		btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	end)
end

createTab("Editor", 0)
createTab("Rota", 100)
createTab("Execucao", 200)

-- Definir aba inicial visível
for k, v in pairs(tabContents) do v.Visible = false end
for k, v in pairs(tabButtons) do
	v.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
	v.TextColor3 = Color3.fromRGB(180, 180, 180)
end
tabContents["Editor"].Visible = true
tabButtons["Editor"].BackgroundColor3 = Color3.fromRGB(0, 200, 120)
tabButtons["Editor"].TextColor3 = Color3.fromRGB(255, 255, 255)

-- Funções auxiliares para visualização ------------------------------------

local function drawPoints(list, modo)
	clearVisuals()
	for i, p in ipairs(list) do
		local color = BrickColor.new("Bright green")
		if modo == "Load" then color = BrickColor.new("Bright blue")
		elseif modo == "Code" then color = BrickColor.new("Bright yellow")
		elseif p.tipo == "Jump" then color = BrickColor.new("Royal purple") end

		if visualMode == "Button" or #list == 1 then
			local part = Instance.new("Part")
			part.Size = Vector3.new(0.5, 0.5, 0.5)
			part.Position = p.pos + Vector3.new(0, 1, 0)
			part.Anchored = true
			part.CanCollide = false
			part.Transparency = 0.4
			part.BrickColor = color
			part.Material = Enum.Material.Neon
			part.Name = "ParkourPointVisual"
			part.Parent = workspace
			table.insert(visuals, part)
		elseif i > 1 then
			local prev = list[i - 1]
			local line = Instance.new("Part")
			local dist = (p.pos - prev.pos).Magnitude
			line.Size = Vector3.new(0.15, 0.15, dist)
			line.CFrame = CFrame.new(prev.pos, p.pos) * CFrame.new(0, 0, -dist / 2)
			line.Anchored = true
			line.CanCollide = false
			line.Material = Enum.Material.Neon
			line.BrickColor = color
			line.Transparency = 0.3
			line.Name = "ParkourLineVisual"
			line.Parent = workspace
			table.insert(visuals, line)
		end
	end
end

local function encode(points)
	local s = ""
	for _, p in ipairs(points) do
		s ..= string.format("%.2f,%.2f,%.2f,%s|", p.pos.X, p.pos.Y, p.pos.Z, p.tipo)
	end
	return s
end

local function decode(str)
	local list = {}
	for x, y, z, tipo in string.gmatch(str, "([%-%.%d]+),([%-%.%d]+),([%-%.%d]+),([%a]+)|") do
		table.insert(list, {
			pos = Vector3.new(tonumber(x), tonumber(y), tonumber(z)),
			tipo = tipo
		})
	end
	return list
end

-- ABA EDITOR ---------------------------------

local editor = tabContents["Editor"]

local editModeBtn = Instance.new("TextButton", editor)
editModeBtn.Size = UDim2.new(1, -20, 0, 30)
editModeBtn.Position = UDim2.new(0, 10, 0, 10)
editModeBtn.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
editModeBtn.Font = Enum.Font.Gotham
editModeBtn.TextSize = 18
editModeBtn.TextColor3 = Color3.new(1, 1, 1)
editModeBtn.Text = "Modo edição: OFF"
editModeBtn.AutoButtonColor = true
editModeBtn.MouseButton1Click:Connect(function(btn)
	editMode = not editMode
	btn.Text = "Modo edição: " .. (editMode and "ON" or "OFF")
end)

local pointTypeBtn = Instance.new("TextButton", editor)
pointTypeBtn.Size = UDim2.new(1, -20, 0, 30)
pointTypeBtn.Position = UDim2.new(0, 10, 0, 50)
pointTypeBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
pointTypeBtn.Font = Enum.Font.Gotham
pointTypeBtn.TextSize = 18
pointTypeBtn.TextColor3 = Color3.new(1, 1, 1)
pointTypeBtn.Text = "Tipo de ponto: Normal"
pointTypeBtn.AutoButtonColor = true
pointTypeBtn.MouseButton1Click:Connect(function(btn)
	if pointType == "Normal" then
		pointType = "Jump"
		btn.Text = "Tipo de ponto: Jump"
	else
		pointType = "Normal"
		btn.Text = "Tipo de ponto: Normal"
	end
end)

local removeLastBtn = Instance.new("TextButton", editor)
removeLastBtn.Size = UDim2.new(1, -20, 0, 30)
removeLastBtn.Position = UDim2.new(0, 10, 0, 90)
removeLastBtn.BackgroundColor3 = Color3.fromRGB(180, 0, 0)
removeLastBtn.Font = Enum.Font.Gotham
removeLastBtn.TextSize = 18
removeLastBtn.TextColor3 = Color3.new(1, 1, 1)
removeLastBtn.Text = "Remover último ponto"
removeLastBtn.AutoButtonColor = true
removeLastBtn.MouseButton1Click:Connect(function()
	if #points > 0 then
		table.remove(points, #points)
		drawPoints(points)
	end
end)

mouse.Button1Down:Connect(function()
	if not editMode then return end
	if mouse.Target then
		local pos = mouse.Hit.Position
		local novoPonto = {pos = pos, tipo = pointType}
		table.insert(points, novoPonto)
		drawPoints(points)
	end
end)

-- ABA ROTA -----------------------------------

local rota = tabContents["Rota"]

local routeNameBox = Instance.new("TextBox", rota)
routeNameBox.Size = UDim2.new(1, -20, 0, 30)
routeNameBox.Position = UDim2.new(0, 10, 0, 10)
routeNameBox.PlaceholderText = "Nome da rota"
routeNameBox.TextColor3 = Color3.new(1, 1, 1)
routeNameBox.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
routeNameBox.ClearTextOnFocus = false

routeNameBox.FocusLost:Connect(function()
	if routeNameBox.Text ~= "" then
		routeName = routeNameBox.Text
	end
end)

local saveRouteBtn = Instance.new("TextButton", rota)
saveRouteBtn.Size = UDim2.new(1, -20, 0, 30)
saveRouteBtn.Position = UDim2.new(0, 10, 0, 60)
saveRouteBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
saveRouteBtn.Font = Enum.Font.Gotham
saveRouteBtn.TextSize = 18
saveRouteBtn.TextColor3 = Color3.new(1, 1, 1)
saveRouteBtn.Text = "Salvar Rota"
saveRouteBtn.AutoButtonColor = true
saveRouteBtn.MouseButton1Click:Connect(function()
	if #points == 0 then return end
	local code = encode(points)
	routes[routeName] = code
	setclipboard(code)
	warn("Código da rota '"..routeName.."' copiado para área de transferência.")
end)

local loadRouteBtn = Instance.new("TextButton", rota)
loadRouteBtn.Size = UDim2.new(1, -20, 0, 30)
loadRouteBtn.Position = UDim2.new(0, 10, 0, 100)
loadRouteBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
loadRouteBtn.Font = Enum.Font.Gotham
loadRouteBtn.TextSize = 18
loadRouteBtn.TextColor3 = Color3.new(1, 1, 1)
loadRouteBtn.Text = "Carregar Rota"
loadRouteBtn.AutoButtonColor = true
loadRouteBtn.MouseButton1Click:Connect(function()
	local code = routes[routeName]
	if not code then
		warn("Nenhuma rota salva com esse nome.")
		return
	end
	points = decode(code)
	drawPoints(points, "Load")
end)

local loadCodeBox = Instance.new("TextBox", rota)
loadCodeBox.Size = UDim2.new(1, -20, 0, 80)
loadCodeBox.Position = UDim2.new(0, 10, 0, 150)
loadCodeBox.PlaceholderText = "Cole o código da rota aqui"
loadCodeBox.TextColor3 = Color3.new(1,1,1)
loadCodeBox.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
loadCodeBox.ClearTextOnFocus = false
loadCodeBox.MultiLine = true

local loadCodeBtn = Instance.new("TextButton", rota)
loadCodeBtn.Size = UDim2.new(1, -20, 0, 30)
loadCodeBtn.Position = UDim2.new(0, 10, 0, 240)
loadCodeBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
loadCodeBtn.Font = Enum.Font.Gotham
loadCodeBtn.TextSize = 18
loadCodeBtn.TextColor3 = Color3.new(1, 1, 1)
loadCodeBtn.Text = "Carregar código"
loadCodeBtn.AutoButtonColor = true
loadCodeBtn.MouseButton1Click:Connect(function()
	local code = loadCodeBox.Text
	if code == "" then return end
	points = decode(code)
	drawPoints(points, "Code")
end)

local visualModeBtn = Instance.new("TextButton", rota)
visualModeBtn.Size = UDim2.new(1, -20, 0, 30)
visualModeBtn.Position = UDim2.new(0, 10, 0, 280)
visualModeBtn.BackgroundColor3 = Color3.fromRGB(0, 255, 150)
visualModeBtn.Font = Enum.Font.Gotham
visualModeBtn.TextSize = 18
visualModeBtn.TextColor3 = Color3.new(1, 1, 1)
visualModeBtn.Text = "Visual: Botão"
visualModeBtn.AutoButtonColor = true
visualModeBtn.MouseButton1Click:Connect(function(btn)
	if visualMode == "Button" then
		visualMode = "Line"
		btn.Text = "Visual: Linha"
	else
		visualMode = "Button"
		btn.Text = "Visual: Botão"
	end
	drawPoints(points)
end)

-- ABA EXECUÇÃO -----------------------------------

local exec = tabContents["Execucao"]

local habLabel = Instance.new("TextLabel", exec)
habLabel.Size = UDim2.new(1, -20, 0, 30)
habLabel.Position = UDim2.new(0, 10, 0, 10)
habLabel.BackgroundTransparency = 1
habLabel.Text = "Habilidade: 0.00"
habLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
habLabel.Font = Enum.Font.Gotham
habLabel.TextSize = 20

local executeBtn = Instance.new("TextButton", exec)
executeBtn.Size = UDim2.new(1, -20, 0, 30)
executeBtn.Position = UDim2.new(0, 10, 0, 60)
executeBtn.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
executeBtn.Font = Enum.Font.Gotham
executeBtn.TextSize = 18
executeBtn.TextColor3 = Color3.new(1, 1, 1)
executeBtn.Text = "Executar Rota"
executeBtn.AutoButtonColor = true

-- Função para criar checkpoint invisível que detecta toque
local function criarCheckpoint(pos)
	local part = Instance.new("Part")
	part.Size = Vector3.new(2, 2, 2)
	part.Position = pos + Vector3.new(
