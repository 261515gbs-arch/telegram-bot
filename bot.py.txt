from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import ApplicationBuilder, CommandHandler, CallbackQueryHandler, MessageHandler, filters, ContextTypes

TOKEN = "8629983125:AAHGub4hHFSq0gMKPnMLj07ng1R5WjjShm8"
ADMIN_ID = 8569191895
WALLET_ADDRESS = "TBHyh15c6bL7NPUDXpnrjp6TGV7tQA7fsy"

user_state = {}
orders = {}

PRICE = {"ghost": 10, "view": 5, "react": 7}
MIN_AMOUNT = {"ghost": 100, "view": 100, "react": 10}
TYPE_NAME = {"ghost": "👻 유령", "view": "👁 뷰", "react": "❤️ 반응"}

def calc_price(t, a):
    return round((PRICE[t] / 1000) * a, 4)

def format_order_message(type_, amount, price):
    return (
        f"✅ 선택 수량: {amount:,}개\n"
        f"💰 금액: {price:.4f} USDT\n"
        f"🔗 링크를 입력해 주세요.\n"
        f"━━━━━━━━━━━━━━\n"
        f"📝 한 줄에 하나씩 입력 (여러 게시물 가능)\n"
        f"예시:\n"
        f"https://t.me/channel/1\n"
        f"https://t.me/channel/2\n"
        f"https://t.me/channel/3"
    )

def split_amount(total, links):
    n = len(links)
    base = total // n
    remainder = total % n
    return [(link, base + (1 if i < remainder else 0)) for i, link in enumerate(links)]

def main_menu(uid):
    if uid == ADMIN_ID:
        return InlineKeyboardMarkup([[InlineKeyboardButton("📦 주문 목록", callback_data="admin_orders")]])
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("🛒 주문하기", callback_data="order")],
        [InlineKeyboardButton("🤖 봇 제작 및 문의", callback_data="support")]
    ])

def order_menu():
    return InlineKeyboardMarkup([
        [InlineKeyboardButton("👻 유령 주문하기", callback_data="ghost")],
        [InlineKeyboardButton("👁 자동 뷰 주문하기", callback_data="view")],
        [InlineKeyboardButton("❤️ 반응 주문하기", callback_data="react")],
        [InlineKeyboardButton("🔙 뒤로가기", callback_data="back_main")]
    ])

def amount_menu(t):
    m = MIN_AMOUNT[t]
    return InlineKeyboardMarkup([
        [InlineKeyboardButton(str(m), callback_data=f"amount_{m}"),
         InlineKeyboardButton(str(m*5), callback_data=f"amount_{m*5}")],
        [InlineKeyboardButton(str(m*10), callback_data=f"amount_{m*10}"),
         InlineKeyboardButton(str(m*50), callback_data=f"amount_{m*50}")],
        [InlineKeyboardButton(str(m*100), callback_data=f"amount_{m*100}")],
        [InlineKeyboardButton("✍️ 직접 입력", callback_data="custom_amount")],
        [InlineKeyboardButton("🔙 뒤로가기", callback_data="back_order")]
    ])

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("메뉴 선택 👇", reply_markup=main_menu(update.message.from_user.id))

async def button(update: Update, context: ContextTypes.DEFAULT_TYPE):
    q = update.callback_query
    await q.answer()
    uid = q.from_user.id

    if q.data == "admin_orders":
        if uid != ADMIN_ID:
            return

        if not orders:
            await q.edit_message_text("📦 주문 없음")
            return

        text = "📦 주문 목록\n\n"

        for user_id, data in orders.items():
            text += (
                f"👤 @{data['username']}\n"
                f"상품: {TYPE_NAME[data['type']]}\n"
                f"수량: {data['amount']}\n"
                f"금액: {data['price']} USDT\n"
                f"링크 수: {len(data['links'])}\n"
                f"━━━━━━━━━━\n"
            )

        await q.edit_message_text(text)
        return

    if q.data == "back_main":
        await q.edit_message_text("메뉴 선택 👇", reply_markup=main_menu(uid)); return

    if q.data == "back_order":
        await q.edit_message_text("주문 종류 선택 👇", reply_markup=order_menu()); return

    if q.data == "order":
        await q.edit_message_text("주문 종류 선택 👇", reply_markup=order_menu()); return

    if q.data in ["ghost", "view", "react"]:
        user_state[uid] = {"type": q.data}
        await q.edit_message_text("수량 선택 👇", reply_markup=amount_menu(q.data)); return

    if q.data.startswith("amount_"):
        amt = int(q.data.split("_")[1])
        t = user_state[uid]["type"]

        if amt < MIN_AMOUNT[t]:
            await q.answer("최소 수량 미만 ❌", show_alert=True); return

        price = calc_price(t, amt)
        user_state[uid].update({"amount": amt, "price": price, "step": "waiting_link"})

        await q.edit_message_text(
            format_order_message(t, amt, price),
            reply_markup=InlineKeyboardMarkup([
                [InlineKeyboardButton("🔙 뒤로가기", callback_data="back_order")]
            ])
        )
        return

    if q.data == "custom_amount":
        user_state[uid]["step"] = "waiting_custom"
        await q.edit_message_text("수량 입력", reply_markup=InlineKeyboardMarkup([
            [InlineKeyboardButton("🔙 뒤로가기", callback_data="back_order")]
        ])); return

    if q.data.startswith("confirm_"):
        if uid != ADMIN_ID: return
        target = int(q.data.split("_")[1])
        data = orders[target]

        split_data = split_amount(data["amount"], data["links"])
        detail = "\n".join([f"{l} → {a}" for l, a in split_data])

        await context.bot.send_message(
            target,
            f"🧾 결제 완료 영수증\n\n상품:{TYPE_NAME[data['type']]}\n총 수량:{data['amount']}\n금액:{data['price']} USDT\n\n📦 분배 결과\n{detail}\n\n작업 시작 🚀"
        )

        await q.edit_message_text("✅ 승인 완료")
        del orders[target]
        return

    if q.data.startswith("reject_"):
        if uid != ADMIN_ID: return
        target = int(q.data.split("_")[1])
        await context.bot.send_message(target, "❌ 주문 취소됨")
        await q.edit_message_text("❌ 거절 완료")
        if target in orders: del orders[target]
        return

    if q.data == "support":
        await q.edit_message_text("관리자 문의: @Ghost_Gala")

async def message_handler(update: Update, context: ContextTypes.DEFAULT_TYPE):
    uid = update.message.from_user.id
    text = update.message.text
    username = update.message.from_user.username
    state = user_state.get(uid)

    if not state and uid not in orders:
        await context.bot.send_message(
            ADMIN_ID,
            f"⚠️ 주문 없는 결제 감지\n\n👤 @{username}\n💰 TXID:{text}\n\n확인 필요 🚨"
        )
        await update.message.reply_text("⚠️ 주문 확인 필요")
        return

    if state and state.get("step") == "waiting_custom":
        if not text.isdigit():
            await update.message.reply_text("숫자만 ❌"); return

        amt = int(text)
        t = state["type"]

        if amt < MIN_AMOUNT[t]:
            await update.message.reply_text(f"최소 {MIN_AMOUNT[t]} 이상 ❌"); return

        price = calc_price(t, amt)
        user_state[uid].update({"amount": amt, "price": price, "step": "waiting_link"})

        await update.message.reply_text(format_order_message(t, amt, price))
        return

    if state and state.get("step") == "waiting_link":
        links = [l.strip() for l in text.split("\n") if l.strip()]

        if not links:
            await update.message.reply_text("링크 없음 ❌")
            return

        orders[uid] = {
            "username": username,
            "type": state["type"],
            "amount": state["amount"],
            "price": state["price"],
            "links": links,
            "txid": None
        }

        split_data = split_amount(state["amount"], links)
        link_text = "\n".join(links)
        split_text = "\n".join([f"{l} → {a}" for l, a in split_data])

        await context.bot.send_message(
            chat_id=ADMIN_ID,
            text=(
                f"🔥 주문\n"
                f"👤 @{username}\n"
                f"📦 상품: {TYPE_NAME[state['type']]}\n"
                f"🔢 총 수량: {state['amount']}\n\n"
                f"🔗 링크 ({len(links)}개)\n{link_text}\n\n"
                f"📦 분배 결과\n{split_text}"
            )
        )

        await update.message.reply_text(
            f"💰 {state['price']} USDT 전송 후 TXID 보내주세요\n\n"
            f"⚠️ 필히 코인주소 확인 부탁드립니다.\n"
            f"⚠️ 오입금시 반환 처리 ❌\n\n"
            f"네트워크: TRC20\n"
            f"주소:\n`{WALLET_ADDRESS}`",
            parse_mode="Markdown"
        )

        user_state[uid]["step"] = "waiting_payment"
        return

    if state and state.get("step") == "waiting_payment":
        orders[uid]["txid"] = text

        keyboard = [[
            InlineKeyboardButton("✅ 승인", callback_data=f"confirm_{uid}"),
            InlineKeyboardButton("❌ 거절", callback_data=f"reject_{uid}")
        ]]

        await context.bot.send_message(
            ADMIN_ID,
            f"💰 입금\n@{username}\n금액:{orders[uid]['price']} USDT\n{text}",
            reply_markup=InlineKeyboardMarkup(keyboard)
        )

        await update.message.reply_text("확인 요청 완료 ✅")
        user_state[uid] = None

app = ApplicationBuilder().token(TOKEN).build()

app.add_handler(CommandHandler("start", start))
app.add_handler(CallbackQueryHandler(button))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, message_handler))

app.run_polling()