local ScreenGui = Instance.new("ScreenGui")
local Main = Instance.new("Frame")
local GridFrame = Instance.new("Frame")
local TopBar = Instance.new("Frame") -- Для красоты и перетаскивания
local Title = Instance.new("TextLabel")
local CloseBtn = Instance.new("TextButton")
local Status = Instance.new("TextLabel")
local FlagBtn = Instance.new("TextButton")
local ResetBtn = Instance.new("TextButton")

-- Защита от повторного запуска (удаляем старый GUI если есть)
if game.CoreGui:FindFirstChild("RealMinesweeper_v2") then
    game.CoreGui.RealMinesweeper_v2:Destroy()
end

ScreenGui.Parent = game.CoreGui
ScreenGui.Name = "RealMinesweeper_v2"
ScreenGui.ResetOnSpawn = false -- Не удаляется при смерти

-- Основной контейнер
Main.Parent = ScreenGui
Main.Size = UDim2.new(0, 320, 0, 410)
Main.Position = UDim2.new(0.5, -160, 0.5, -205)
Main.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true -- В Delta это обычно работает лучше всего

-- Верхняя панель
TopBar.Parent = Main
TopBar.Size = UDim2.new(1, 0, 0, 30)
TopBar.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
TopBar.BorderSizePixel = 0

Title.Parent = TopBar
Title.Size = UDim2.new(1, -40, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.Text = "Minesweeper Delta"
Title.TextColor3 = Color3.fromRGB(200, 200, 200)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.BackgroundTransparency = 1
Title.Font = Enum.Font.SourceSansBold
Title.TextSize = 16

CloseBtn.Parent = TopBar
CloseBtn.Size = UDim2.new(0, 30, 0, 30)
CloseBtn.Position = UDim2.new(1, -30, 0, 0)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.fromRGB(255, 100, 100)
CloseBtn.BackgroundTransparency = 1
CloseBtn.TextSize = 18
CloseBtn.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)

Status.Parent = Main
Status.Size = UDim2.new(1, 0, 0, 30)
Status.Position = UDim2.new(0, 0, 0, 35)
Status.BackgroundTransparency = 1
Status.TextColor3 = Color3.new(1, 1, 1)
Status.Font = Enum.Font.Code
Status.TextSize = 18

GridFrame.Parent = Main
GridFrame.Position = UDim2.new(0, 10, 0, 70)
GridFrame.Size = UDim2.new(0, 300, 0, 300)
GridFrame.BackgroundTransparency = 1

-- Параметры игры
local size = 8
local bombCount = 10
local grid = {}
local buttons = {}
local isFlagMode = false
local gameOver = false
local tilesRevealed = 0

local function checkWin()
    -- Победа если: (Всего клеток - Бомбы) == Открытые клетки
    if tilesRevealed == (size * size - bombCount) then
        gameOver = true
        Status.Text = "🎉 ПОБЕДА! 🎉"
        Status.TextColor3 = Color3.new(0, 1, 0)
        -- Показываем где были бомбы (не взрывая)
        for i = 1, #grid do
            if grid[i].isBomb then
                buttons[i].Text = "💣"
                buttons[i].BackgroundColor3 = Color3.new(0, 0.8, 0) -- Зеленые бомбы
            end
        end
    end
end

local function reveal(x, y)
    local idx = (y - 1) * size + x
    -- Проверки границ и состояний
    if x < 1 or x > size or y < 1 or y > size then return end
    if grid[idx].revealed or grid[idx].flagged or gameOver then return end
    
    local cell = grid[idx]
    local btn = buttons[idx]
    
    cell.revealed = true
    tilesRevealed = tilesRevealed + 1
    
    if cell.isBomb then
        btn.BackgroundColor3 = Color3.new(0.8, 0, 0)
        btn.Text = "💥"
        gameOver = true
        Status.Text = "Game Over!"
        Status.TextColor3 = Color3.new(1, 0, 0)
        -- Показать остальные бомбы
        for i = 1, #grid do
            if grid[i].isBomb and not grid[i].revealed then
                buttons[i].Text = "💣"
                buttons[i].BackgroundColor3 = Color3.fromRGB(100, 50, 50)
            end
        end
    else
        btn.BackgroundColor3 = Color3.fromRGB(180, 180, 180) -- Светлый фон для открытой
        if cell.near > 0 then
            btn.Text = tostring(cell.near)
            -- Цвета цифр как в оригинале
            local c = {Color3.fromRGB(0,0,255), Color3.fromRGB(0,128,0), Color3.fromRGB(255,0,0), Color3.fromRGB(0,0,128)}
            btn.TextColor3 = c[cell.near] or Color3.new(0,0,0)
            btn.Font = Enum.Font.SourceSansBold
        else
            btn.Text = "" -- Пусто
            -- Безопасная рекурсия (Flood Fill)
            for dy = -1, 1 do
                for dx = -1, 1 do
                    if not (dx == 0 and dy == 0) then
                        reveal(x + dx, y + dy)
                    end
                end
            end
        end
        checkWin()
    end
end

local function initGame()
    gameOver = false
    tilesRevealed = 0
    Status.Text = "Бомб осталось: " .. bombCount
    Status.TextColor3 = Color3.new(1, 1, 1)
    GridFrame:ClearAllChildren()
    grid = {}
    buttons = {}
    
    -- Инициализация данных
    for i = 1, size * size do 
        table.insert(grid, {isBomb = false, revealed = false, flagged = false, near = 0}) 
    end
    
    -- Расстановка бомб
    local placed = 0
    while placed < bombCount do
        local r = math.random(1, size * size)
        if not grid[r].isBomb then
            grid[r].isBomb = true
            placed = placed + 1
        end
    end
    
    -- Подсчет соседей
    for y = 1, size do
        for x = 1, size do
            local idx = (y - 1) * size + x
            if not grid[idx].isBomb then
                local n = 0
                for dy = -1, 1 do
                    for dx = -1, 1 do
                        local nx, ny = x + dx, y + dy
                        if nx >= 1 and nx <= size and ny >= 1 and ny <= size then
                            local nIdx = (ny - 1) * size + nx
                            if grid[nIdx].isBomb then n = n + 1 end
                        end
                    end
                end
                grid[idx].near = n
            end
        end
    end
    
    -- Создание кнопок
    local cellSize = 300 / size
    for y = 1, size do
        for x = 1, size do
            local idx = (y - 1) * size + x
            local b = Instance.new("TextButton", GridFrame)
            b.Size = UDim2.new(0, cellSize - 2, 0, cellSize - 2)
            b.Position = UDim2.new(0, (x-1)*cellSize, 0, (y-1)*cellSize)
            b.BackgroundColor3 = Color3.fromRGB(80, 80, 90)
            b.Text = ""
            b.TextSize = 20
            b.TextColor3 = Color3.new(1,1,1)
            
            b.MouseButton1Click:Connect(function()
                if gameOver then return end
                if isFlagMode then
                    if not grid[idx].revealed then
                        grid[idx].flagged = not grid[idx].flagged
                        b.Text = grid[idx].flagged and "🚩" or ""
                        b.TextColor3 = Color3.new(1, 0.5, 0)
                    end
                else
                    reveal(x, y)
                end
            end)
            buttons[idx] = b
        end
    end
end

-- Нижние кнопки управления
FlagBtn.Parent = Main
FlagBtn.Size = UDim2.new(0, 120, 0, 30)
FlagBtn.Position = UDim2.new(0, 20, 1, -35)
FlagBtn.Text = "⛏️ Копать"
FlagBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 60)
FlagBtn.TextColor3 = Color3.new(1,1,1)
FlagBtn.MouseButton1Click:Connect(function()
    isFlagMode = not isFlagMode
    if isFlagMode then
        FlagBtn.Text = "🚩 Флаг"
        FlagBtn.BackgroundColor3 = Color3.fromRGB(180, 80, 0)
    else
        FlagBtn.Text = "⛏️ Копать"
        FlagBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 60)
    end
end)

ResetBtn.Parent = Main
ResetBtn.Size = UDim2.new(0, 120, 0, 30)
ResetBtn.Position = UDim2.new(1, -140, 1, -35)
ResetBtn.Text = "Заново"
ResetBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
ResetBtn.TextColor3 = Color3.new(1,1,1)
ResetBtn.MouseButton1Click:Connect(initGame)

initGame()
