<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
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

	async function loadTasks() {
		try {
			const allTasks = await db.table<Task>('tasks').toArray();
			tasks.set(allTasks); // Обновляем Svelte Store
		} catch (error) {
			console.error('Ошибка при загрузке задач:', error);
		}
	}

	async function handleSubmit() {
		if (!newTaskTitle.trim()) {
			alert('Название задачи не может быть пустым!');
			return;
		}

		if (editingTask) {
			try {
				await db.table('tasks').update(editingTask.id!, {
					title: newTaskTitle.trim(),
					description: newTaskDescription.trim()
				});
				console.log('Задача обновлена успешно!');
			} catch (error) {
				console.error('Ошибка при обновлении задачи:', error);
			} finally {
				editingTask = null; // Сбрасываем режим редактирования
			}
		} else {
			const task: Task = {
				title: newTaskTitle.trim(),
				description: newTaskDescription.trim(),
				status: 'active' // Новая задача всегда активна
			};
			try {
				await db.table('tasks').add(task);
				console.log('Задача добавлена успешно!');
			} catch (error) {
				console.error('Ошибка при добавлении задачи:', error);
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


		if (confirm('Вы уверены, что хотите удалить эту задачу?')) {
			try {
				await db.table('tasks').delete(id);
				console.log('Задача удалена успешно!');
				await loadTasks(); // Перезагружаем задачи после удаления
			} catch (error) {
				console.error('Ошибка при удалении задачи:', error);
			}
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

<div class="min-h-screen bg-gray-100 p-4 sm:p-6 lg:p-8 flex flex-col items-center">
	<div class="w-full max-w-4xl  rounded-lg p-6 sm:p-8">
		<h1 class="text-3xl sm:text-4xl font-bold text-center text-gray-800 mb-8">Мой список задач</h1>

		<!-- Форма добавления/редактирования задачи -->
		<form on:submit|preventDefault={handleSubmit} class="mb-8 p-4 sm:p-6 bg-white rounded-lg shadow-md">
			<div class="mb-4">
				<label for="title" class="block text-gray-700 text-sm font-bold mb-2">Название задачи:</label>
				<input
					type="text"
					id="title"
					bind:value={newTaskTitle}
					placeholder="Введите название задачи"
					class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-blue-500 transition duration-200 ease-in-out"
					required
				/>
			</div>
			<div class="mb-6">
				<label for="description" class="block text-gray-700 text-sm font-bold mb-2">Описание:</label>
				<textarea
					id="description"
					bind:value={newTaskDescription}
					placeholder="Введите описание задачи (необязательно)"
					rows="3"
					class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline focus:border-blue-500 transition duration-200 ease-in-out resize-y"
				></textarea>
			</div>
			<button
				type="submit"
				class="bg-blue-600 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform w-full sm:w-auto"
			>
				{editingTask ? 'Сохранить изменения' : 'Добавить задачу'}
			</button>
		</form>

		<!-- Список задач -->
		{#if $tasks.length === 0}
			<p class="text-center text-gray-500 text-lg">Задач пока нет. Добавьте первую!</p>
		{:else}
			<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
				{#each $tasks as task (task.id)}
					<div class="bg-white p-4 rounded-lg shadow-lg flex flex-col justify-between border border-gray-200">
						<div>
							<h3 class="text-xl font-semibold text-gray-800 mb-2">{task.title}</h3>
							{#if task.description}
								<p class="text-gray-600 text-sm mb-2">{task.description}</p>
							{/if}
							<span
								class="text-xs font-medium px-2 py-1 rounded-full {task.status === 'active' ? 'bg-yellow-100 text-yellow-800' : 'bg-green-100 text-green-800'}">
                Статус: {task.status === 'active' ? 'Активная' : 'Завершена'}
              </span>
						</div>
						<div class="mt-4 flex flex-col sm:flex-row gap-2">
							<button
								on:click={() => editTask(task)}
								class="bg-emerald-700 hover:bg-emerald-900 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform flex-grow"
							>
								Редактировать
							</button>
							<button
								on:click={() => deleteTask(task.id)}
								class="bg-red-700 hover:bg-red-900 text-white font-bold py-2 px-4 rounded-lg shadow-md transition duration-300 ease-in-out transform  flex-grow"
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