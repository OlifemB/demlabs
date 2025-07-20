<script lang="ts">
	// Импорт необходимых модулей Svelte
	import { onMount } from 'svelte';
	import { writable, derived } from 'svelte/store';
	import { flip } from 'svelte/animate'; // Импорт для анимации

	// Импорт библиотеки Dexie для работы с IndexDB
	import Dexie from 'dexie';

	// Определение интерфейса для задачи
	interface Task {
		id?: number; // ID будет генерироваться автоматически Dexie
		title: string;
		description?: string;
		status: 'active' | 'completed';
	}

	// Создание экземпляра базы данных Dexie
	// 'todoDatabase' - название базы данных
	// 'tasks' - название таблицы (store)
	// '++id' - автоинкрементный первичный ключ
	// '&title' - уникальный индекс по полю title (для поиска)
	// 'status' - индекс по полю status
	const db = new Dexie('todoDatabase');
	db.version(1).stores({
		tasks: '++id, title, description, status' // Определение схемы таблицы
	});

	// Инициализация базы данных начальными данными, если она пуста
	db.on('populate', async () => {
		await db.table('tasks').bulkAdd([
			{ title: 'Купить продукты', description: 'Молоко, хлеб, яйца', status: 'active' },
			{ title: 'Завершить отчет', description: 'Подготовить презентацию для встречи', status: 'active' },
			{ title: 'Позвонить другу', description: 'Узнать как дела', status: 'completed' }
		]);
	});

	// Svelte Store для хранения списка задач
	const tasks = writable<Task[]>([]);

	// Переменные для полей формы
	let newTaskTitle: string = '';
	let newTaskDescription: string = '';
	let editingTask: Task | null = null; // Хранит задачу, которая сейчас редактируется

	// Переменная для поискового запроса, теперь это Svelte Store
	const searchQuery = writable<string>('');

	// Derived store для фильтрации задач по поисковому запросу
	const filteredTasks = derived(
		[tasks, searchQuery], // Зависимости: основной список задач и поисковый запрос
		([$tasks, $searchQuery], set) => {
			const lowerCaseQuery = $searchQuery.toLowerCase().trim();
			if (!lowerCaseQuery) {
				set($tasks); // Если запрос пуст, показываем все задачи
			} else {
				// Фильтруем задачи по названию или описанию (без учета регистра)
				const filtered = $tasks.filter(task =>
					task.title.toLowerCase().includes(lowerCaseQuery) ||
					(task.description && task.description.toLowerCase().includes(lowerCaseQuery))
				);
				set(filtered);
			}
		}
	);

	// Переменные для кастомного модального окна
	let showModal: boolean = false;
	let modalMessage: string = '';
	let isConfirmModal: boolean = false;
	let confirmAction: (() => void) | null = null; // Функция, которая будет выполнена при подтверждении

	// Флаг состояния загрузки для скелетона
	let isLoading: boolean = true;

	// Функция для отображения кастомного алерта
	function showAlert(message: string) {
		modalMessage = message;
		isConfirmModal = false;
		showModal = true;
	}

	// Функция для отображения кастомного окна подтверждения
	function showConfirm(message: string, action: () => void) {
		modalMessage = message;
		isConfirmModal = true;
		confirmAction = action;
		showModal = true;
	}

	// Функция для закрытия модального окна
	function closeModal() {
		showModal = false;
		modalMessage = '';
		isConfirmModal = false;
		confirmAction = null;
	}

	// Функция для подтверждения действия в модальном окне
	function handleConfirm() {
		if (confirmAction) {
			confirmAction();
		}
		closeModal();
	}

	// Функция для загрузки задач из IndexDB
	async function loadTasks() {
		isLoading = true; // Устанавливаем флаг загрузки в true
		try {
			const allTasks = await db.table<Task>('tasks').toArray();
			tasks.set(allTasks); // Обновляем Svelte Store
		} catch (error) {
			console.error('Ошибка при загрузке задач:', error);
			showAlert('Ошибка при загрузке задач. Пожалуйста, попробуйте позже.');
		} finally {
			isLoading = false; // Устанавливаем флаг загрузки в false после завершения
		}
	}

	// Функция для добавления или обновления задачи
	async function handleSubmit() {
		// Проверяем, что название задачи не пустое
		if (!newTaskTitle.trim()) {
			showAlert('Название задачи не может быть пустым!');
			return;
		}

		if (editingTask) {
			// Режим редактирования
			try {
				await db.table('tasks').update(editingTask.id!, {
					title: newTaskTitle.trim(),
					description: newTaskDescription.trim(),
				});
				console.log('Задача обновлена успешно!');
				showAlert('Задача успешно обновлена!');
			} catch (error) {
				console.error('Ошибка при обновлении задачи:', error);
				showAlert('Ошибка при обновлении задачи. Пожалуйста, попробуйте позже.');
			} finally {
				editingTask = null; // Сбрасываем режим редактирования
			}
		} else {
			// Режим добавления новой задачи
			const task: Task = {
				title: newTaskTitle.trim(),
				description: newTaskDescription.trim(),
				status: 'active', // Новая задача всегда активна
			};
			try {
				await db.table('tasks').add(task);
				console.log('Задача добавлена успешно!');
				showAlert('Задача успешно добавлена!');
			} catch (error) {
				console.error('Ошибка при добавлении задачи:', error);
				showAlert('Ошибка при добавлении задачи. Пожалуйста, попробуйте позже.');
			}
		}

		// Очищаем форму и перезагружаем задачи
		newTaskTitle = '';
		newTaskDescription = '';
		await loadTasks();
	}

	// Функция для перехода в режим редактирования
	function editTask(task: Task) {
		editingTask = task;
		newTaskTitle = task.title;
		newTaskDescription = task.description || '';
	}

	// Функция для удаления задачи
	async function deleteTask(id: number | undefined) {
		if (id === undefined) return; // Проверка на undefined ID

		// Используем кастомное окно подтверждения
		showConfirm('Вы уверены, что хотите удалить эту задачу?', async () => {
			try {
				await db.table('tasks').delete(id);
				console.log('Задача удалена успешно!');
				showAlert('Задача успешно удалена!');
				await loadTasks(); // Перезагружаем задачи после удаления
			} catch (error) {
				console.error('Ошибка при удалении задачи:', error);
				showAlert('Ошибка при удалении задачи. Пожалуйста, попробуйте позже.');
			}
		});
	}

	// Загружаем задачи при монтировании компонента
	onMount(async () => {
		await loadTasks();
	});
</script>

<style>
    /* Убедимся, что шрифт Inter используется по умолчанию */
    body {
        font-family: 'Inter', sans-serif;
    }
</style>

<!-- Основной контейнер с фоном и центрированием -->
<div class="min-h-screen bg-gray-200 p-4 sm:p-6 lg:p-8 flex flex-col items-center justify-center">
	<!-- Главный контейнер приложения с мягкой тенью и скруглениями -->
	<div class="w-full max-w-4xl p-6 sm:p-8">
		<h1 class="text-3xl sm:text-4xl font-extrabold text-center text-gray-800 mb-8 drop-shadow-sm">Мой список задач</h1>

		<!-- Форма добавления/редактирования задачи с мягкой тенью и скруглениями -->
		<form on:submit|preventDefault={handleSubmit} class="mb-8 p-4 sm:p-6 bg-white rounded-2xl shadow-lg border border-gray-200 transition duration-300 ease-in-out hover:shadow-xl">
			<div class="mb-4">
				<label for="title" class="block text-gray-700 text-sm font-semibold mb-2">Название задачи:</label>
				<input
					type="text"
					id="title"
					bind:value={newTaskTitle}
					placeholder="Введите название задачи"
					class="shadow-sm appearance-none border border-gray-200 rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-200 ease-in-out"
					required
				/>
			</div>
			<div class="mb-6">
				<label for="description" class="block text-gray-700 text-sm font-semibold mb-2">Описание:</label>
				<textarea
					id="description"
					bind:value={newTaskDescription}
					placeholder="Введите описание задачи (необязательно)"
					rows="3"
					class="shadow-sm appearance-none border border-gray-200 rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-200 ease-in-out resize-y"
				></textarea>
			</div>
			<button
				type="submit"
				class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-xl shadow-md transition duration-300 ease-in-out w-full sm:w-auto focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-opacity-75"
			>
				{editingTask ? 'Сохранить изменения' : 'Добавить задачу'}
			</button>
		</form>

		<!-- Поле поиска -->
		<div class="mb-8 p-4 sm:p-6 bg-white rounded-2xl shadow-lg border border-gray-200">
			<label for="search" class="block text-gray-700 text-sm font-semibold mb-2">Поиск задач:</label>
			<input
				type="text"
				id="search"
				bind:value={$searchQuery}
				placeholder="Искать по названию или описанию..."
				class="shadow-sm appearance-none border border-gray-200 rounded-lg w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:ring-2 focus:ring-blue-400 focus:border-transparent transition duration-200 ease-in-out"
			/>
		</div>

		<!-- Список задач -->
		{#if isLoading}
			<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
				{#each Array(4) as _, i (i)}
					<div class="bg-white p-4 rounded-2xl shadow-lg flex flex-col justify-between border border-gray-200 animate-pulse">
						<div class="h-6 bg-gray-300 rounded w-3/4 mb-2"></div>
						<div class="h-4 bg-gray-300 rounded w-full mb-2"></div>
						<div class="h-4 bg-gray-300 rounded w-1/2 mb-4"></div>
						<div class="h-6 bg-gray-300 rounded w-1/4 mb-2"></div>
						<div class="mt-4 flex flex-col sm:flex-row gap-2">
							<div class="h-10 bg-gray-300 rounded-xl flex-grow"></div>
							<div class="h-10 bg-gray-300 rounded-xl flex-grow"></div>
						</div>
					</div>
				{/each}
			</div>
		{:else if $filteredTasks.length === 0}
			<p class="text-center text-gray-500 text-lg py-10">
				{#if $searchQuery}
					Задач по вашему запросу не найдено.
				{:else}
					Задач пока нет. Добавьте первую!
				{/if}
			</p>
		{:else}
			<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
				{#each $filteredTasks as task (task.id)}
					<!-- Карточка задачи с мягкой тенью и микровзаимодействиями -->
					<div
						animate:flip={{ duration: 300 }}
						class="bg-white p-4 rounded-2xl shadow-lg flex flex-col justify-between border border-gray-200 transition duration-300 ease-in-out hover:shadow-xl"
					>
						<div>
							<h3 class="text-xl font-semibold text-gray-800 mb-2">{task.title}</h3>
							{#if task.description}
								<p class="text-gray-600 text-sm mb-2">{task.description}</p>
							{/if}
							<!-- Акцент на цвете статуса -->
							<span class="text-xs font-bold px-3 py-1 rounded-full {task.status === 'active' ? 'bg-yellow-200 text-yellow-800' : 'bg-green-200 text-green-800'} shadow-sm">
                Статус: {task.status === 'active' ? 'Активная' : 'Завершена'}
              </span>
						</div>
						<div class="mt-4 flex flex-col sm:flex-row gap-2">
							<button
								on:click={() => editTask(task)}
								class="bg-green-700 hover:bg-green-900 text-white font-bold py-2 px-4 rounded-xl shadow-md transition duration-300 ease-in-out flex-grow focus:outline-none focus:ring-2 focus:ring-green-400 focus:ring-opacity-75"
							>
								Редактировать
							</button>
							<button
								on:click={() => deleteTask(task.id)}
								class="bg-red-700 hover:bg-red-900 text-white font-bold py-2 px-4 rounded-xl shadow-md transition duration-300 ease-in-out flex-grow focus:outline-none focus:ring-2 focus:ring-red-400 focus:ring-opacity-75"
							>
								Удалить
							</button>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>

<!-- Кастомное модальное окно -->
{#if showModal}
	<div class="fixed inset-0 bg-gray-900 bg-opacity-50 flex items-center justify-center p-4 z-50">
		<div class="bg-white rounded-lg shadow-xl p-6 sm:p-8 max-w-sm w-full text-center">
			<p class="text-lg font-semibold text-gray-800 mb-6">{modalMessage}</p>
			{#if isConfirmModal}
				<div class="flex justify-center gap-4">
					<button
						on:click={handleConfirm}
						class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-opacity-75"
					>
						Да
					</button>
					<button
						on:click={closeModal}
						class="bg-gray-300 hover:bg-gray-400 text-gray-800 font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out focus:outline-none focus:ring-2 focus:ring-gray-400 focus:ring-opacity-75"
					>
						Отмена
					</button>
				</div>
			{:else}
				<button
					on:click={closeModal}
					class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out focus:outline-none focus:ring-2 focus:ring-blue-400 focus:ring-opacity-75"
				>
					ОК
				</button>
			{/if}
		</div>
	</div>
{/if}
