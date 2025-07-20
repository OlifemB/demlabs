<script lang="ts">
	import { onMount } from 'svelte';
	import { writable, derived } from 'svelte/store';
	import { flip } from 'svelte/animate';
	import { fly, fade } from 'svelte/transition';

	import Dexie from 'dexie';

	interface Task {
		id?: number;
		title: string;
		description?: string;
		status: 'active' | 'completed';
	}

	const db = new Dexie('todoDatabase');
	db.version(1).stores({
		tasks: '++id, title, description, status'
	});

	db.on('populate', async () => {
		await db.table('tasks').bulkAdd([
			{ title: 'Купить продукты', description: 'Молоко, хлеб, яйца', status: 'active' },
			{ title: 'Завершить отчет', description: 'Подготовить презентацию для встречи', status: 'active' },
			{ title: 'Позвонить другу', description: 'Узнать как дела', status: 'completed' }
		]);
	});

	const tasks = writable<Task[]>([]);

	let newTaskTitle: string = '';
	let newTaskDescription: string = '';
	let editingTask: Task | null = null;

	const searchQuery = writable<string>('');

	const filteredTasks = derived(
		[tasks, searchQuery],
		([$tasks, $searchQuery], set) => {
			const lowerCaseQuery = $searchQuery.toLowerCase().trim();
			if (!lowerCaseQuery) {
				set($tasks);
			} else {
				const filtered = $tasks.filter(task =>
					task.title.toLowerCase().includes(lowerCaseQuery) ||
					(task.description && task.description.toLowerCase().includes(lowerCaseQuery))
				);
				set(filtered);
			}
		}
	);

	let isLoading: boolean = true;

	const notifications = writable<{ id: number; message: string; type: 'success' | 'error' }[]>([]);
	let notificationIdCounter = 0;

	function addNotification(message: string, type: 'success' | 'error') {
		const id = notificationIdCounter++;
		notifications.update(n => [...n, { id, message, type }]);
		setTimeout(() => {
			notifications.update(n => n.filter(notification => notification.id !== id));
		}, 2000);
	}

	async function loadTasks() {
		isLoading = true;
		try {
			const allTasks = await db.table<Task>('tasks').toArray();
			tasks.set(allTasks);
		} catch (error) {
			console.error('Ошибка при загрузке задач:', error);
			addNotification('Ошибка при загрузке задач. Пожалуйста, попробуйте позже.', 'error');
		} finally {
			isLoading = false;
		}
	}

	async function handleSubmit() {
		if (!newTaskTitle.trim()) {
			addNotification('Название задачи не может быть пустым!', 'error');
			return;
		}

		if (editingTask) {
			try {
				await db.table('tasks').update(editingTask.id!, {
					title: newTaskTitle.trim(),
					description: newTaskDescription.trim(),
				});
				console.log('Задача обновлена успешно!');
				addNotification('Задача успешно обновлена!', 'success');
			} catch (error) {
				console.error('Ошибка при обновлении задачи:', error);
				addNotification('Ошибка при обновлении задачи. Пожалуйста, попробуйте позже.', 'error');
			} finally {
				editingTask = null;
			}
		} else {
			const task: Task = {
				title: newTaskTitle.trim(),
				description: newTaskDescription.trim(),
				status: 'active',
			};
			try {
				await db.table('tasks').add(task);
				console.log('Задача добавлена успешно!');
				addNotification('Задача успешно добавлена!', 'success');
			} catch (error) {
				console.error('Ошибка при добавлении задачи:', error);
				addNotification('Ошибка при добавлении задачи. Пожалуйста, попробуйте позже.', 'error');
			}
		}

		newTaskTitle = '';
		newTaskDescription = '';
		await loadTasks();
	}

	function editTask(task: Task) {
		editingTask = task;
		newTaskTitle = task.title;
		newTaskDescription = task.description || '';
	}

	async function deleteTask(id: number | undefined) {
		if (id === undefined) return;

		try {
			await db.table('tasks').delete(id);
			console.log('Задача удалена успешно!');
			addNotification('Задача успешно удалена!', 'success');
			await loadTasks();
		} catch (error) {
			console.error('Ошибка при удалении задачи:', error);
			addNotification('Ошибка при удалении задачи. Пожалуйста, попробуйте позже.', 'error');
		}
	}

	onMount(async () => {
		await loadTasks();
	});
</script>

<style>
    body {
        font-family: 'Inter', sans-serif;
    }
</style>

<div class="min-h-screen bg-gray-200 p-4 sm:p-6 lg:p-8 flex flex-col items-center justify-start">
	<div class="w-full max-w-4xl p-6 sm:p-8">
		<h1 class="text-3xl sm:text-4xl font-extrabold text-center text-gray-800 mb-8 drop-shadow-sm">Мой список задач</h1>

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
					<div
						animate:flip={{ duration: 300 }}
						class="bg-white p-4 rounded-2xl shadow-lg flex flex-col justify-between border border-gray-200 transition duration-300 ease-in-out hover:shadow-xl"
					>
						<div>
							<h3 class="text-xl font-semibold text-gray-800 mb-2">{task.title}</h3>
							{#if task.description}
								<p class="text-gray-600 text-sm mb-2">{task.description}</p>
							{/if}
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

<div class="fixed top-4 right-4 z-50 flex flex-col space-y-2">
	{#each $notifications as notification (notification.id)}
		<div
			class="p-4 rounded-lg shadow-lg text-white {notification.type === 'success' ? 'bg-green-700' : 'bg-red-700'} bg-opacity-80 transition-opacity duration-300 ease-in-out"
			in:fly="{{ y: -20, duration: 300 }}"
			out:fade="{{ duration: 300 }}"
		>
			{notification.message}
		</div>
	{/each}
</div>
