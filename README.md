!pip install python-telegram-bot==13.13 sympy matplotlib chempy
import logging
from telegram import Update
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters, CallbackContext
import sympy as sp
from chempy import balance_stoichiometry
import matplotlib.pyplot as plt
import io
import os

# Configure logging
logging.basicConfig(level=logging.INFO)

# Hardcoded token for simplicity (replace with your actual token)
TOKEN = ''

# Start command handler
def start(update: Update, context: CallbackContext):
    context.bot.send_message(chat_id=update.effective_chat.id, text="Hello! I'm a math and chemistry bot. Ask me a question or input an equation.")

# Message handler for various commands
def handle_message(update: Update, context: CallbackContext):
    message = update.message.text.lower()
    if 'solve' in message:
        equation = message.split('solve')[1].strip()
        solution = solve_equation(equation)
        context.bot.send_message(chat_id=update.effective_chat.id, text=solution)
    elif 'balance' in message:
        try:
            reaction = message.split('balance')[1].strip()
            solution = handle_chemistry(reaction)
            context.bot.send_message(chat_id=update.effective_chat.id, text=solution)
        except Exception as e:
            context.bot.send_message(chat_id=update.effective_chat.id, text=f"Error: {e}")
    elif 'plot' in message:
        equation = message.split('plot')[1].strip()
        img = plot_graph(equation)
        if isinstance(img, str):
            context.bot.send_message(chat_id=update.effective_chat.id, text=img)
        else:
            context.bot.send_photo(chat_id=update.effective_chat.id, photo=img)
    else:
        context.bot.send_message(chat_id=update.effective_chat.id, text="I didn't understand that. Please try again.")

# Preprocess equations for sympy
def preprocess_equation(equation):
    equation = equation.replace('^', '**')  # Replace caret with double asterisk for exponentiation
    return equation

# Solve mathematical equations
def solve_equation(equation):
    try:
        equation = preprocess_equation(equation)
        x = sp.symbols('x')
        equation = sp.sympify(equation)
        steps = []

        # Factor the equation
        factored = sp.factor(equation)
        steps.append(f"Factored form: {sp.pretty(factored)}")

        # Solve the equation
        solutions = sp.solve(equation, x)
        steps.append(f"Solutions: {solutions}")

        # Create detailed steps for each part of the solving process
        detailed_steps = "\n".join(steps)
        return detailed_steps
    except Exception as e:
        return f"Error: {e}"

# Handle chemical reaction balancing
def handle_chemistry(reaction):
    try:
        # Check if the reaction contains the '->' symbol
        if '->' not in reaction:
            return "Error: Please provide a valid reaction in the form 'reactants -> products'."

        # Split the reaction into reactants and products
        reactants, products = reaction.split('->')

        # Ensure reactants and products are not empty
        if not reactants.strip() or not products.strip():
            return "Error: Reaction must have both reactants and products."

        # Split reactants and products into sets
        reactants_set = set(reactants.strip().split('+'))
        products_set = set(products.strip().split('+'))

        # Balance the stoichiometry
        reac, prod = balance_stoichiometry(reactants_set, products_set)

        # Format the balanced reaction
        balanced_reactants = ' + '.join(f'{v} {k}' for k, v in reac.items())
        balanced_products = ' + '.join(f'{v} {k}' for k, v in prod.items())
        balanced_equation = f"{balanced_reactants} -> {balanced_products}"

        # Return the balanced equation
        return f"Balanced equation: {balanced_equation}"
    except Exception as e:
        return f"Error: {e}"

# Plot graphs for equations
def plot_graph(expression):
    try:
        expression = preprocess_equation(expression)
        x = sp.symbols('x')
        expr = sp.sympify(expression)
        p = sp.plot(expr, show=False)
        img = io.BytesIO()
        p.save(img)
        img.seek(0)
        return img
    except Exception as e:
        return f"Error: {e}"

# Main function to start the bot
def main():
    updater = Updater(TOKEN, use_context=True)
    dp = updater.dispatcher

    dp.add_handler(CommandHandler('start', start))
    dp.add_handler(MessageHandler(Filters.text & ~Filters.command, handle_message))

    updater.start_polling()
    updater.idle()

if __name__ == '__main__':
    main()

