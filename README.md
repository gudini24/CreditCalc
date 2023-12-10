# CreditCalc
import logging
import re
from aiogram import Bot, Dispatcher, types, executor
from aiogram.contrib.fsm_storage.memory import MemoryStorage
from aiogram.dispatcher import FSMContext
from aiogram.dispatcher.filters import Command
from aiogram.dispatcher.filters.state import State, StatesGroup

# Устанавливаем уровень логов для отладки
logging.basicConfig(level=logging.INFO)

# Создаем объекты бота и диспетчера
bot = Bot(token="11111111111111111111")
storage = MemoryStorage()
dp = Dispatcher(bot, storage=storage)

# Создаем класс-состояние для хранения информации о пользователе
class LoanCalculatorStates(StatesGroup):
    waiting_loan_term = State()
    waiting_car_cost = State()
    waiting_interest_rate = State()
    waiting_down_payment = State()

# Функция для рассчета ежемесячных платежей
def calculate_monthly_payment(principal, interest_rate, duration):
    r = interest_rate / 100 / 12
    n = duration
    monthly_payment = (principal * r * (1 + r)**n) / ((1 + r)**n - 1)
    return monthly_payment

# Обработчик команды /start
@dp.message_handler(commands=["start"])
async def cmd_start(message: types.Message):
    # Отправляем приветственное сообщение и инструкции
    await message.answer("Привет! Я помогу тебе рассчитать ежемесячные платежи по автокредиту.\n\n"
                         "Введите срок кредита в месяцах:")

    # Устанавливаем первоначальное состояние - ожидание срока кредита
    await LoanCalculatorStates.waiting_loan_term.set()

@dp.message_handler(state=LoanCalculatorStates.waiting_loan_term)
async def process_loan_term(message: types.Message, state: FSMContext):
    # Получаем введенный пользователем срок кредита
    loan_term = int(re.sub(r'\D', '', message.text))

    # Сохраняем полученные данные в состоянии FSM
    await state.update_data(loan_term=loan_term)

    # Запрашиваем следующие данные - стоимость автомобиля
    await message.answer("Введите стоимость автомобиля:")

    # Устанавливаем следующее состояние - ожидание стоимости автомобиля
    await LoanCalculatorStates.waiting_car_cost.set()

@dp.message_handler(state=LoanCalculatorStates.waiting_car_cost)
async def process_car_cost(message: types.Message, state: FSMContext):
    # Получаем введенную пользователем стоимость автомобиля
    car_cost = float(re.sub(r'[^\d.]', '', message.text))

    # Сохраняем полученные данные в состоянии FSM
    await state.update_data(car_cost=car_cost)

    # Запрашиваем следующие данные - процентная ставка
    await message.answer("Введите процентную ставку:")

    # Устанавливаем следующее состояние - ожидание процентной ставки
    await LoanCalculatorStates.waiting_interest_rate.set()

@dp.message_handler(state=LoanCalculatorStates.waiting_interest_rate)
async def process_interest_rate(message: types.Message, state: FSMContext):
    # Получаем введенную пользователем процентную ставку
    interest_rate = float(re.sub(r'[^\d.]', '', message.text))

    # Сохраняем полученные данные в состоянии FSM
    await state.update_data(interest_rate=interest_rate)

    # Запрашиваем следующие данные - первоначальный взнос
    await message.answer("Введите первоначальный взнос:")

    # Устанавливаем следующее состояние - ожидание первоначального взноса
    await LoanCalculatorStates.waiting_down_payment.set()

@dp.message_handler(state=LoanCalculatorStates.waiting_down_payment)
async def process_down_payment(message: types.Message, state: FSMContext):
    # Получаем введенный пользователем первоначальный взнос
    down_payment = float(re.sub(r'[^\d.]', '', message.text))

    # Получаем все сохраненные данные из состояния FSM
    data = await state.get_data()
    loan_term = int(data["loan_term"])
    car_cost = float(data["car_cost"])
    interest_rate = float(data["interest_rate"])

    # Выполняем расчет ежемесячного платежа
    principal = car_cost - down_payment
    monthly_payment = calculatemonthlypayment(principal, interestrate, loanterm)

       # Создаем таблицу для вывода ежемесячных платежей
    table = "{:<20} {:<20} {:<20} {:<20}\n".format("Остаток основного долга", "Проценты", "Основной долг", "Ежемесячный платеж кредита:")
    remaining_principal = principal

    for month in range(1, loan_term + 1):
            interest = remaining_principal * (interest_rate / 12 / 100)
            principal_payment = monthly_payment - interest
            remaining_principal -= principal_payment

            table += "{:<20.2f} {:<20.2f} {:<20.2f} {:<20.2f}\n".format(remaining_principal, interest, principal_payment, monthly_payment)

        # Отправляем результат пользователю
    await message.answer(table)
# Добавляем обработчик для кнопки "Сбросить"
@dp.callback_query_handler(text="reset")
async def btn_reset(callback_query: types.CallbackQuery, state: FSMContext):
    # Сбрасываем состояние FSM
    await state.finish()

    # Отправляем сообщение с просьбой ввести данные заново
    await callback_query.message.answer("Предыдущий запрос сброшен. Введите срок кредита в месяцах:")

    # Устанавливаем первоначальное состояние - ожидание срока кредита
    await LoanCalculatorStates.waiting_loan_term.set()

# ...

# В обработчиках каждого шага добавляем inline-клавиатуру с кнопкой "Сбросить"
@dp.message_handler(state=LoanCalculatorStates.waiting_loan_term)
async def process_loan_term(message: types.Message, state: FSMContext):
    # ...

    # Создаем inline-клавиатуру
    keyboard = types.InlineKeyboardMarkup()
    keyboard.add(types.InlineKeyboardButton("Сбросить", callback_data="reset"))
# Запускаем бота
if __name__ == 'main':
    executor.start_polling(dp, skip_updates=True)
