"""
Financial Freedom OS — ONE FILE
Includes: Full platform + Money Education + Stripe Payments (Monthly + Lifetime)

RUN LOCALLY:
  python -m venv .venv
  source .venv/bin/activate
  pip install -r requirements.txt
  python financial_freedom_os.py

ENV VARS (set in Render dashboard):
  SECRET_KEY=...
  DATABASE_URL=...              (auto-set by Render Postgres)
  PREMIUM_MASTER_KEY=...
  STRIPE_SECRET_KEY=sk_live_...
  STRIPE_PUBLISHABLE_KEY=pk_live_...
  STRIPE_WEBHOOK_SECRET=whsec_...
  STRIPE_MONTHLY_PRICE_ID=price_...
  STRIPE_LIFETIME_PRICE_ID=price_...
  APP_URL=https://your-app.onrender.com
"""

from __future__ import annotations
import os, csv, io, json, datetime as dt, secrets, hashlib
from dataclasses import dataclass
from typing import Optional, List, Dict, Tuple
from flask import (Flask, request, redirect, url_for, session,
                   flash, abort, render_template_string, jsonify)
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash

# ── Stripe (optional — gracefully disabled if not configured) ─────────────────
try:
    import stripe
    STRIPE_ENABLED = bool(os.environ.get("STRIPE_SECRET_KEY"))
    if STRIPE_ENABLED:
        stripe.api_key = os.environ.get("STRIPE_SECRET_KEY")
except ImportError:
    STRIPE_ENABLED = False

STRIPE_PK            = os.environ.get("STRIPE_PUBLISHABLE_KEY", "")
STRIPE_WEBHOOK_SECRET= os.environ.get("STRIPE_WEBHOOK_SECRET", "")
STRIPE_MONTHLY_PRICE = os.environ.get("STRIPE_MONTHLY_PRICE_ID", "")
STRIPE_LIFETIME_PRICE= os.environ.get("STRIPE_LIFETIME_PRICE_ID", "")
APP_URL              = os.environ.get("APP_URL", "http://localhost:5000")

# ── Config ────────────────────────────────────────────────────────────────────
def normalize_db_url(url):
    return url.replace("postgres://", "postgresql://", 1) if url.startswith("postgres://") else url

DATABASE_URL       = normalize_db_url(os.environ.get("DATABASE_URL", "sqlite:///ffos.db"))
SECRET_KEY         = os.environ.get("SECRET_KEY", "dev_secret_change_me")
PREMIUM_MASTER_KEY = os.environ.get("PREMIUM_MASTER_KEY", "CHANGE_ME_NOW")

app = Flask(__name__)
app.config["SECRET_KEY"] = SECRET_KEY
app.config["SQLALCHEMY_DATABASE_URI"] = DATABASE_URL
app.config["SQLALCHEMY_TRACK_MODIFICATIONS"] = False
db = SQLAlchemy(app)

# ── Models ────────────────────────────────────────────────────────────────────

class User(db.Model):
    id               = db.Column(db.Integer, primary_key=True)
    email            = db.Column(db.String(180), unique=True, nullable=False)
    password_hash    = db.Column(db.String(255), nullable=False)
    created_at       = db.Column(db.DateTime, default=dt.datetime.utcnow)
    is_premium       = db.Column(db.Boolean, default=False)
    premium_key      = db.Column(db.String(80), nullable=True)
    # Stripe fields
    stripe_customer_id    = db.Column(db.String(80), nullable=True)
    stripe_subscription_id= db.Column(db.String(80), nullable=True)
    subscription_status   = db.Column(db.String(30), nullable=True)  # active/canceled/lifetime
    # Org
    org_id    = db.Column(db.Integer, db.ForeignKey("organization.id"), nullable=True)
    org_role  = db.Column(db.String(20), default="member")

    def set_password(self, pw):   self.password_hash = generate_password_hash(pw)
    def check_password(self, pw): return check_password_hash(self.password_hash, pw)
    @property
    def handle(self): return self.email.split("@")[0]

class Organization(db.Model):
    id          = db.Column(db.Integer, primary_key=True)
    name        = db.Column(db.String(120), nullable=False)
    seat_limit  = db.Column(db.Integer, default=10)
    created_at  = db.Column(db.DateTime, default=dt.datetime.utcnow)
    invite_code = db.Column(db.String(40), unique=True, nullable=False)
    users       = db.relationship("User", backref="org", lazy=True)

class Profile(db.Model):
    id                       = db.Column(db.Integer, primary_key=True)
    user_id                  = db.Column(db.Integer, db.ForeignKey("user.id"), unique=True, nullable=False)
    display_name             = db.Column(db.String(80), nullable=False, default="Household")
    monthly_income           = db.Column(db.Float, default=0.0)
    fixed_bills              = db.Column(db.Float, default=0.0)
    variable_spend           = db.Column(db.Float, default=0.0)
    debt_minimums            = db.Column(db.Float, default=0.0)
    emergency_fund_target    = db.Column(db.Float, default=500.0)
    monthly_investing_target = db.Column(db.Float, default=100.0)
    extra_debt_target        = db.Column(db.Float, default=0.0)
    emergency_fund_current   = db.Column(db.Float, default=0.0)
    created_at               = db.Column(db.DateTime, default=dt.datetime.utcnow)

class Transaction(db.Model):
    id          = db.Column(db.Integer, primary_key=True)
    user_id     = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    date        = db.Column(db.Date, nullable=False)
    description = db.Column(db.String(240), nullable=False)
    amount      = db.Column(db.Float, nullable=False)
    category    = db.Column(db.String(60), nullable=False, default="Uncategorized")
    created_at  = db.Column(db.DateTime, default=dt.datetime.utcnow)

class Debt(db.Model):
    id              = db.Column(db.Integer, primary_key=True)
    user_id         = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    name            = db.Column(db.String(120), nullable=False)
    balance         = db.Column(db.Float, nullable=False, default=0.0)
    apr             = db.Column(db.Float, nullable=False, default=0.0)
    minimum_payment = db.Column(db.Float, nullable=False, default=0.0)
    created_at      = db.Column(db.DateTime, default=dt.datetime.utcnow)

class CommunityPost(db.Model):
    id         = db.Column(db.Integer, primary_key=True)
    user_id    = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    created_at = db.Column(db.DateTime, default=dt.datetime.utcnow)
    title      = db.Column(db.String(120), nullable=False)
    body       = db.Column(db.Text, nullable=False)

class PremiumKey(db.Model):
    id              = db.Column(db.Integer, primary_key=True)
    key             = db.Column(db.String(80), unique=True, nullable=False)
    is_used         = db.Column(db.Boolean, default=False)
    used_by_user_id = db.Column(db.Integer, nullable=True)
    created_at      = db.Column(db.DateTime, default=dt.datetime.utcnow)

class EduLesson(db.Model):
    __tablename__ = "edu_lesson"
    id           = db.Column(db.Integer, primary_key=True)
    lesson_key   = db.Column(db.String(10), unique=True, nullable=False)
    audience     = db.Column(db.String(10), nullable=False, default="adult")
    module       = db.Column(db.String(80), nullable=False, default="Foundations")
    title        = db.Column(db.String(160), nullable=False)
    tagline      = db.Column(db.String(200), nullable=False, default="")
    icon         = db.Column(db.String(8), nullable=False, default="📚")
    color        = db.Column(db.String(10), nullable=False, default="#00e87a")
    content_json = db.Column(db.Text, nullable=False)
    stat         = db.Column(db.String(200), nullable=False, default="")
    action       = db.Column(db.String(400), nullable=False, default="")
    age_group    = db.Column(db.String(40), nullable=True)
    order_index  = db.Column(db.Integer, default=1)
    is_premium   = db.Column(db.Boolean, default=False)
    questions    = db.relationship("EduQuizQuestion", backref="lesson", lazy=True,
                                   order_by="EduQuizQuestion.id", cascade="all, delete-orphan")

class EduQuizQuestion(db.Model):
    __tablename__ = "edu_quiz_question"
    id          = db.Column(db.Integer, primary_key=True)
    lesson_id   = db.Column(db.Integer, db.ForeignKey("edu_lesson.id"), nullable=False)
    prompt      = db.Column(db.String(300), nullable=False)
    opt_a       = db.Column(db.String(200), nullable=False)
    opt_b       = db.Column(db.String(200), nullable=False)
    opt_c       = db.Column(db.String(200), nullable=False)
    opt_d       = db.Column(db.String(200), nullable=False)
    correct     = db.Column(db.String(1), nullable=False)
    explanation = db.Column(db.String(500), nullable=True)

class EduProgress(db.Model):
    __tablename__ = "edu_progress"
    id         = db.Column(db.Integer, primary_key=True)
    user_id    = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    lesson_id  = db.Column(db.Integer, db.ForeignKey("edu_lesson.id"), nullable=False)
    completed  = db.Column(db.Boolean, default=False)
    score      = db.Column(db.Integer, default=0)
    updated_at = db.Column(db.DateTime, default=dt.datetime.utcnow)
    __table_args__ = (db.UniqueConstraint("user_id", "lesson_id", name="uq_edu_user_lesson"),)


class PasswordResetToken(db.Model):
    __tablename__ = "password_reset_token"
    id         = db.Column(db.Integer, primary_key=True)
    user_id    = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    token_hash = db.Column(db.String(64), unique=True, nullable=False)
    expires_at = db.Column(db.DateTime, nullable=False)
    used       = db.Column(db.Boolean, default=False)
    created_at = db.Column(db.DateTime, default=dt.datetime.utcnow)

class OnboardingState(db.Model):
    __tablename__ = "onboarding_state"
    id         = db.Column(db.Integer, primary_key=True)
    user_id    = db.Column(db.Integer, db.ForeignKey("user.id"), unique=True, nullable=False)
    completed  = db.Column(db.Boolean, default=False)
    step       = db.Column(db.Integer, default=1)
    updated_at = db.Column(db.DateTime, default=dt.datetime.utcnow)

class ScoreHistory(db.Model):
    __tablename__ = "score_history"
    id          = db.Column(db.Integer, primary_key=True)
    user_id     = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    score       = db.Column(db.Integer, nullable=False)
    recorded_at = db.Column(db.DateTime, default=dt.datetime.utcnow)


class Lesson(db.Model):
    id           = db.Column(db.Integer, primary_key=True)
    title        = db.Column(db.String(160), nullable=False)
    module       = db.Column(db.String(80), nullable=False, default="Foundations")
    level        = db.Column(db.String(40), nullable=False, default="All")
    content      = db.Column(db.Text, nullable=False)
    order_index  = db.Column(db.Integer, default=1)
    is_published = db.Column(db.Boolean, default=True)
    created_at   = db.Column(db.DateTime, default=dt.datetime.utcnow)
    questions    = db.relationship("QuizQuestion", backref="lesson", lazy=True, order_by="QuizQuestion.id")

class QuizQuestion(db.Model):
    id          = db.Column(db.Integer, primary_key=True)
    lesson_id   = db.Column(db.Integer, db.ForeignKey("lesson.id"), nullable=False)
    prompt      = db.Column(db.String(300), nullable=False)
    a           = db.Column(db.String(200), nullable=False)
    b           = db.Column(db.String(200), nullable=False)
    c           = db.Column(db.String(200), nullable=False)
    d           = db.Column(db.String(200), nullable=False)
    correct     = db.Column(db.String(1), nullable=False)
    explanation = db.Column(db.String(500), nullable=True)

class LessonProgress(db.Model):
    id         = db.Column(db.Integer, primary_key=True)
    user_id    = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    lesson_id  = db.Column(db.Integer, db.ForeignKey("lesson.id"), nullable=False)
    completed  = db.Column(db.Boolean, default=False)
    score      = db.Column(db.Integer, default=0)
    updated_at = db.Column(db.DateTime, default=dt.datetime.utcnow)
    __table_args__ = (db.UniqueConstraint("user_id", "lesson_id", name="uq_user_lesson"),)

class BudgetPlan(db.Model):
    id                   = db.Column(db.Integer, primary_key=True)
    user_id              = db.Column(db.Integer, db.ForeignKey("user.id"), nullable=False)
    month                = db.Column(db.String(7), nullable=False)
    total_income_planned = db.Column(db.Float, default=0.0)
    notes                = db.Column(db.String(240), default="")
    created_at           = db.Column(db.DateTime, default=dt.datetime.utcnow)
    categories           = db.relationship("BudgetCategoryPlan", backref="plan", lazy=True, cascade="all, delete-orphan")
    __table_args__ = (db.UniqueConstraint("user_id", "month", name="uq_user_month"),)

class BudgetCategoryPlan(db.Model):
    id             = db.Column(db.Integer, primary_key=True)
    budget_id      = db.Column(db.Integer, db.ForeignKey("budget_plan.id"), nullable=False)
    category       = db.Column(db.String(60), nullable=False)
    planned_amount = db.Column(db.Float, default=0.0)

# ── Education Seed ────────────────────────────────────────────────────────────

EDU_SEED = [
  dict(lesson_key="A1",audience="adult",module="Foundations",order_index=1,icon="☕",color="#f5a623",is_premium=False,
    title="The Latte Factor",tagline="Small daily leaks steal your future wealth",
    content=["The Latte Factor is a mental model, not a war on coffee. It represents the unconscious daily spending that compounds into hundreds of thousands of dollars of lost wealth over a lifetime.",
      "The math: $10/day in small purchases × 365 days = $3,650/year. Invested at 7% over 40 years = $780,000+. That's the real cost of unawareness.",
      "Your assignment: Track every purchase under $15 for one week. Write it down. The awareness alone changes behavior. Most people find $300–$600/month in spending they don't even remember making.",
      "Key reframe: This isn't about deprivation. It's about making conscious choices instead of automatic ones. Spend on what you love. Cut what you don't notice."],
    stat="$10/day invested × 40 years = $780K+",action="Track every purchase under $15 for 7 days. Find your Latte Factor.",
    questions=[dict(prompt="What is the primary purpose of identifying your Latte Factor?",opt_a="To stop drinking coffee",opt_b="To become aware of unconscious daily spending",opt_c="To reduce all discretionary spending",opt_d="To impress financial advisors",correct="B",explanation="The Latte Factor is about awareness, not deprivation."),
      dict(prompt="At 7% annual return, how much does $10/day grow to over 40 years?",opt_a="$146,000",opt_b="$350,000",opt_c="$780,000",opt_d="$1,200,000",correct="C",explanation="Compound interest on consistent daily savings creates dramatic long-term wealth.")]),
  dict(lesson_key="A2",audience="adult",module="Foundations",order_index=2,icon="🤖",color="#00e87a",is_premium=False,
    title="Pay Yourself First — Automate It",tagline="Make saving automatic before willpower fails",
    content=["Every financial system that relies on willpower eventually fails. The solution is automation. Before you pay any bill, a percentage of your income moves automatically to savings and investments.",
      "The government already does this to you — payroll taxes are taken before you see your check. Apply the same principle to your own wealth.",
      "How to implement: Set up an automatic transfer the same day your paycheck hits. Even 5% to start. Then increase by 1% every 3 months until you're at 15–20%.",
      "The psychological trick: You adjust your lifestyle to what's left. Automation removes the temptation to spend first entirely."],
    stat="Automating 10% of $60K salary = $1M+ by retirement",action="Set up one automatic transfer today — even $25. The habit matters more than the amount.",
    questions=[dict(prompt="Why is automation more effective than willpower for saving?",opt_a="Automation earns higher interest",opt_b="Willpower is unlimited for most people",opt_c="You adjust lifestyle to what remains after saving",opt_d="Banks require it",correct="C",explanation="People spend what's available. Automation removes the choice."),
      dict(prompt="When should you set up your automatic savings transfer?",opt_a="End of the month with what's left",opt_b="Same day your paycheck arrives",opt_c="Quarterly",opt_d="After all bills are paid",correct="B",explanation="Pay yourself first means the transfer happens before anything else.")]),
  dict(lesson_key="A3",audience="adult",module="Debt",order_index=3,icon="🎯",color="#e85d75",is_premium=True,
    title="The DOLP Debt System",tagline="Destroy debt in the mathematically correct order",
    content=["DOLP stands for Debt On Lowest Principal. It's a hybrid system that balances mathematical efficiency with behavioral psychology.",
      "How to calculate: Take your current balance and divide it by your minimum monthly payment. Example: $4,200 ÷ $85 minimum = DOLP of 49.4.",
      "Rank all debts from lowest DOLP to highest. Pay minimums on everything. Every extra dollar attacks the lowest DOLP debt first.",
      "Why this works: DOLP accounts for how quickly you can eliminate each account — creating wins while remaining relatively efficient."],
    stat="Average American carries $6,500 in credit card debt at 20%+ APR",action="List all debts. Calculate DOLP for each. Circle the one to attack first.",
    questions=[dict(prompt="How do you calculate a debt's DOLP number?",opt_a="APR ÷ Balance",opt_b="Balance ÷ Minimum Payment",opt_c="Minimum Payment × 12",opt_d="Balance × APR",correct="B",explanation="DOLP = Balance ÷ Minimum Payment. Lower DOLP = attack it first."),
      dict(prompt="After paying off your lowest DOLP debt, what do you do?",opt_a="Spend it as a reward",opt_b="Save it in your emergency fund",opt_c="Roll it to the next debt on the list",opt_d="Invest it immediately",correct="C",explanation="Rolling payments creates a debt snowball effect.")]),
  dict(lesson_key="A4",audience="adult",module="Investing",order_index=4,icon="📈",color="#7b68ee",is_premium=True,
    title="The 401(k) Match: Never Leave Free Money",tagline="Employer match = instant 50–100% return",
    content=["If your employer offers a 401(k) match and you are not contributing enough to get the full match, you are leaving part of your compensation on the table.",
      "Example: Employer matches 50% up to 6% of salary. $60K salary contributing 6% ($3,600) = employer adds $1,800 — an instant 50% return before the market does anything.",
      "Contribution order: (1) 401(k) to the match. (2) HSA if eligible. (3) Roth IRA to max. (4) Back to 401(k) annual limit. (5) Taxable brokerage.",
      "Choose low-cost index funds inside the 401(k) — expense ratio under 0.20%. Avoid high-fee actively managed funds."],
    stat="50% match on 6% contribution = instant 50% ROI before any market gains",action="Log in to HR portal today. Verify you're contributing enough to get the full employer match.",
    questions=[dict(prompt="What is the correct first step in the investment order of operations?",opt_a="Max out Roth IRA",opt_b="Open a brokerage account",opt_c="Contribute to 401(k) up to the employer match",opt_d="Pay off all debt first",correct="C",explanation="Employer match is an immediate guaranteed return. Always capture it first."),
      dict(prompt="What type of fund should you prioritize inside your 401(k)?",opt_a="High-fee actively managed funds",opt_b="Individual company stocks",opt_c="Low-cost index funds under 0.20% expense ratio",opt_d="Bond funds only",correct="C",explanation="Low-cost index funds minimize fees that compound against you over a career.")]),
  dict(lesson_key="A5",audience="adult",module="Investing",order_index=5,icon="⏰",color="#2a8c4e",is_premium=True,
    title="Start Late? Start NOW",tagline="Every month of delay costs more than you think",
    content=["The most damaging financial behavior is waiting to start. At 7% returns, $500/month started at 40 grows to ~$600K by 70. Started at 30, same amount: ~$1.2M.",
      "If you're starting late: (1) Increase savings rate aggressively — 20%+ if possible. (2) Use catch-up contributions if over 50. (3) Extend working years by 2–3 years. (4) Reduce expenses to free up capital.",
      "Cognitive trap: 'I'm too far behind, so why bother.' This is the most expensive thought in personal finance. Starting imperfectly today beats starting perfectly never.",
      "The honest math: A 50-year-old investing $1,000/month at 7% for 20 years accumulates $520,000. Real money. Real options. Real security."],
    stat="Delaying 10 years costs approximately 50% of your final portfolio value",action="Calculate your current savings rate. If under 15%, increase by 1% this month.",
    questions=[dict(prompt="What is the most damaging financial behavior?",opt_a="Investing in index funds",opt_b="Waiting to start investing",opt_c="Using a 401(k)",opt_d="Paying off debt before investing",correct="B",explanation="Delay is the most expensive financial decision most people make."),
      dict(prompt="Which is a valid lever for someone starting to invest late?",opt_a="Avoid investing until debt is zero",opt_b="Lower savings rate to maintain lifestyle",opt_c="Use catch-up contributions if over 50",opt_d="Wait for the market to dip",correct="C",explanation="The IRS allows higher contribution limits for people 50+. Use every tool available.")]),
  dict(lesson_key="A6",audience="adult",module="Protection",order_index=6,icon="🛡️",color="#2a7a6b",is_premium=True,
    title="Protection Before Optimization",tagline="One emergency can undo years of wealth-building",
    content=["You cannot build wealth if a single crisis wipes you out. Protection infrastructure must come before aggressive investing.",
      "Emergency fund: 3–6 months of essential expenses in a high-yield savings account. Not in the market. Not in your checking account. Liquid, boring, stable.",
      "Insurance checklist: (1) Term life insurance if anyone depends on you — 10–12x annual income. (2) Disability insurance. (3) Health insurance — one hospitalization without it is catastrophic.",
      "The sequence: Starter emergency fund ($1,000) → Employer match → High-interest debt → Full emergency fund → Max tax-advantaged accounts → Taxable investing."],
    stat="60% of personal bankruptcies are caused by medical bills — even among insured people",action="Do you have 3 months of expenses saved? If not, that's your #1 priority.",
    questions=[dict(prompt="Where should your emergency fund be held?",opt_a="Invested in index funds",opt_b="In your regular checking account",opt_c="In a high-yield savings account — liquid and stable",opt_d="In a CD locked for 2 years",correct="C",explanation="Emergency funds must be liquid and stable. HYSAs offer both plus interest."),
      dict(prompt="How much term life insurance is generally recommended?",opt_a="$100,000 flat",opt_b="1–2x annual income",opt_c="10–12x annual income",opt_d="Whatever your employer provides",correct="C",explanation="10–12x annual income ensures dependents can replace your income for an extended period.")]),
  dict(lesson_key="K1",audience="kids",module="Money Basics",order_index=1,icon="🐷",color="#e85d75",is_premium=False,age_group="Ages 5–10",
    title="The Three Jar System",tagline="Every dollar has a job",
    content=["Whenever you get money — from allowance, chores, or birthday gifts — split it into three jars: SAVE, SPEND, and GIVE.",
      "SAVE jar: This money is for big dreams. A new game, a bike, something you really want but can't buy today.",
      "SPEND jar: This is for small fun things you want right now — a snack, a small toy. It's YOUR money to enjoy.",
      "GIVE jar: Giving money to help others makes you feel rich in a different way. The happiest people in the world are givers."],
    stat="Even splitting $1 into three parts teaches the most important money habit",action="Find three jars. Label them SAVE · SPEND · GIVE. Start today!",
    questions=[dict(prompt="What are the three jars in the Three Jar System?",opt_a="Earn, Borrow, Spend",opt_b="Save, Spend, Give",opt_c="Banks, Stocks, Cash",opt_d="Food, Fun, School",correct="B",explanation="Save · Spend · Give — every dollar goes into one of these three."),
      dict(prompt="What is the GIVE jar for?",opt_a="Saving up for expensive things",opt_b="Daily snacks and small purchases",opt_c="Helping others or causes you care about",opt_d="Paying for school supplies",correct="C",explanation="Giving is a powerful habit that makes you feel abundant and helps others.")]),
  dict(lesson_key="K2",audience="kids",module="Money Basics",order_index=2,icon="⏳",color="#f5a623",is_premium=False,age_group="Ages 8–13",
    title="The Magic of Waiting",tagline="Your money makes more money — just by waiting",
    content=["Here's the coolest secret in all of money: if you put your savings in the right place, it GROWS by itself. You just have to wait.",
      "This is called interest. The bank pays you for leaving your money there. Then they pay you interest on your interest too — like a snowball rolling downhill.",
      "Example: Put $100 in savings at 5%. After 1 year: $105. After 10 years: $163. After 30 years: $432. Your $100 turned into $432 — without doing ANYTHING!",
      "This is why starting young is a SUPERPOWER. The earlier you start, the more time your snowball has to roll."],
    stat="$100 saved at age 10 could be worth $3,000+ by retirement",action="Ask a parent to help you find a savings account. Watch it grow every month!",
    questions=[dict(prompt="What is interest?",opt_a="Money you have to pay back",opt_b="A bank fee",opt_c="Money the bank pays you for keeping your savings there",opt_d="A type of credit card",correct="C",explanation="Interest is money the bank pays YOU for letting them hold your savings."),
      dict(prompt="Why is starting to save young such a big advantage?",opt_a="Banks give special rates to kids",opt_b="More time for compound interest to grow your money",opt_c="Adults are not allowed to save",opt_d="Schools pay you to save",correct="B",explanation="Compound interest needs TIME. Starting at 10 vs 30 can mean hundreds of thousands of dollars difference!")]),
  dict(lesson_key="K3",audience="kids",module="Money Basics",order_index=3,icon="🧠",color="#7b68ee",is_premium=False,age_group="Ages 6–12",
    title="Needs vs. Wants",tagline="The question rich people ask before every purchase",
    content=["Before buying ANYTHING, wealthy people ask one question: Is this a NEED or a WANT?",
      "NEEDS are things you truly must have: food, shelter, clothes for school, medicine, transportation.",
      "WANTS are things that are fun and nice to have, but you could live without: video games, candy, the newest sneakers when your current ones still work.",
      "Knowing the difference doesn't mean you never buy WANTS. It means you CHOOSE to buy them on purpose — and that feels way better than impulse buying."],
    stat="The 24-hour rule: Wait a day before buying a want. Most impulse desires fade quickly.",action="Next time you want to buy something, ask: NEED or WANT? Then decide on purpose.",
    questions=[dict(prompt="Which of these is a NEED?",opt_a="New video game",opt_b="Candy at the checkout",opt_c="School lunch",opt_d="A second pair of sneakers",correct="C",explanation="School lunch is food — a genuine need. The others are wants."),
      dict(prompt="What is the 24-hour rule?",opt_a="Spend money within 24 hours of getting it",opt_b="Wait a day before buying a want to see if you still want it",opt_c="Save for 24 months before spending",opt_d="Only shop on certain days",correct="B",explanation="Waiting 24 hours separates impulse buying from intentional spending.")]),
  dict(lesson_key="K4",audience="kids",module="Earning",order_index=4,icon="💼",color="#2a8c4e",is_premium=True,age_group="Ages 7–14",
    title="Money Comes From Value",tagline="The more you help people, the more you can earn",
    content=["Here's how ALL money works: someone pays you because you did something useful for them.",
      "When you do chores, you help your family. When you mow a lawn, you solve a neighbor's problem. Every business on earth works exactly this way.",
      "The big secret of business: find a problem people have, solve it better than anyone else, and people will gladly pay you.",
      "Your challenge: Brainstorm 5 things you could do to help someone. Dog walking, yard work, tutoring, baking. Pick one and try it this week!"],
    stat="Many millionaires started their first business before age 12",action="List 3 things you could do to earn money this week. Pick one and do it!",
    questions=[dict(prompt="Why do people pay you money?",opt_a="Because they feel sorry for you",opt_b="Because banks print extra money",opt_c="Because you did something useful or valuable for them",opt_d="Because of luck",correct="C",explanation="All money is exchanged for value. The more helpful you become, the more you can earn."),
      dict(prompt="What is the basic secret of all business?",opt_a="Spend as little as possible",opt_b="Find a problem people have and solve it",opt_c="Always charge the highest price",opt_d="Work as many hours as possible",correct="B",explanation="Every successful business solves a real problem for real people.")]),
  dict(lesson_key="K5",audience="kids",module="Saving",order_index=5,icon="🎯",color="#e67e22",is_premium=True,age_group="Ages 6–13",
    title="Save With a Goal",tagline="Goals make saving exciting instead of boring",
    content=["Saving money with no target is boring. Saving toward something you LOVE is exciting. Always know exactly what you're saving for.",
      "How to set a savings goal: (1) Pick something you really want. (2) Find out exactly how much it costs. (3) Divide by how much you save each week. That's your countdown!",
      "Example: You want a $60 game. You save $5/week. 60 ÷ 5 = 12 weeks. Mark a countdown calendar. Cross off each week.",
      "Pro tip: Put a picture of your goal on your SAVE jar. Every time you add money, look at the picture. That image keeps you from spending the savings on something else."],
    stat="Buying something you saved for feels 10x better than buying on impulse",action="Choose ONE thing you want. Find the price. Calculate how many weeks to save for it.",
    questions=[dict(prompt="What is the best way to make saving feel exciting?",opt_a="Save as much as possible with no specific purpose",opt_b="Have a clear goal and know exactly what you're saving toward",opt_c="Ask your parents to save for you",opt_d="Put all money in a bank and forget it",correct="B",explanation="A specific goal makes every dollar feel like progress."),
      dict(prompt="You want an $80 item and save $8 per week. How many weeks?",opt_a="8 weeks",opt_b="80 weeks",opt_c="10 weeks",opt_d="16 weeks",correct="C",explanation="$80 ÷ $8/week = 10 weeks. Always calculate your timeline.")]),
  dict(lesson_key="K6",audience="kids",module="Giving",order_index=6,icon="❤️",color="#c8860a",is_premium=True,age_group="Ages 5–12",
    title="Giving Makes You Richer",tagline="The happiest people are always givers",
    content=["This sounds backwards, but it's true: people who give money away regularly are happier AND tend to build more wealth over their lifetime.",
      "When you give, your brain releases dopamine — the same chemical that makes you feel happy when something great happens. Giving literally makes you feel good.",
      "You don't have to be rich to give. Even $1 to a cause you care about counts. Find something that matters to you.",
      "Giving trains your brain to feel ABUNDANT instead of SCARED about money. Abundant thinking leads to better decisions throughout life."],
    stat="Studies show regular givers report higher life satisfaction and make better financial decisions",action="Choose one cause you care about. Add it to your GIVE jar. Fill it up — then donate!",
    questions=[dict(prompt="What does giving money do to your brain?",opt_a="Makes you feel poorer",opt_b="Has no effect",opt_c="Releases dopamine — the happiness chemical",opt_d="Causes stress",correct="C",explanation="Giving activates reward centers in the brain. It's scientifically proven to increase happiness."),
      dict(prompt="What mindset does regular giving help develop?",opt_a="A scarcity mindset",opt_b="An abundant mindset — feeling there is enough to share",opt_c="A competitive mindset",opt_d="A spending mindset",correct="B",explanation="Giving trains your brain to feel abundant rather than fearful.")]),

  # ── WEALTH SECRETS MODULE (Tax Strategies) ─────────────────────────────────
  dict(lesson_key="T1",audience="adult",module="Wealth Secrets",order_index=7,icon="🏥",color="#22c98a",is_premium=False,
    title="The HSA Triple Tax Advantage",tagline="The only triple tax-free account in existence",
    content=["A Health Savings Account (HSA) is the single most powerful tax-advantaged account available to everyday Americans — and most people use it wrong. They treat it as a spending account instead of a wealth-building vehicle.",
      "The triple advantage: (1) Contributions are pre-tax — reduces your taxable income dollar for dollar. (2) Growth inside the account is completely tax-free. (3) Withdrawals for qualified medical expenses are tax-free. No other account does all three.",
      "The wealth strategy the rich use: Pay all medical expenses out of pocket while you're young and healthy. Let the HSA grow invested in index funds for decades. After age 65, you can withdraw for ANY reason — just like a traditional IRA.",
      "2024 contribution limits: $4,150 for individuals, $8,300 for families. To qualify you must have a High Deductible Health Plan (HDHP). Check your employer's benefits — many offer HSA-compatible plans."],
    stat="HSA invested over 30 years at 7% = $315,000+ completely tax-free",action="Check if your health plan is HSA-eligible. If yes, open an HSA and invest it — don't just save cash in it.",
    questions=[dict(prompt="What makes the HSA a 'triple' tax advantage?",opt_a="It works in three countries",opt_b="Pre-tax contributions, tax-free growth, and tax-free medical withdrawals",opt_c="You get three accounts",opt_d="Three family members can contribute",correct="B",explanation="HSA contributions reduce taxable income, grow tax-free, and withdrawals for medical expenses are never taxed."),
      dict(prompt="What is the wealth-building strategy for HSAs?",opt_a="Spend it immediately on medical bills",opt_b="Keep it all in cash",opt_c="Invest it and pay medical expenses out of pocket to let it grow",opt_d="Withdraw it annually",correct="C",explanation="Treating the HSA as an investment account rather than a spending account creates powerful tax-free wealth.")]),

  dict(lesson_key="T2",audience="adult",module="Wealth Secrets",order_index=8,icon="💸",color="#22c98a",is_premium=False,
    title="The 0% Capital Gains Rate",tagline="Most middle-class families qualify — and don't know it",
    content=["This is one of the most overlooked tax benefits in the entire tax code. If your taxable income is below a certain threshold, you pay ZERO federal tax on long-term investment gains. Zero.",
      "2024 thresholds: Single filers under ~$47,025 pay 0% on long-term capital gains. Married filing jointly under ~$94,050 pay 0%. This includes profits from selling stocks, ETFs, mutual funds, and real estate held over one year.",
      "How families use this: In a low-income year (job change, early retirement, sabbatical), strategically sell appreciated investments and realize gains at 0%. Then rebuy. Your cost basis resets with no tax owed.",
      "This is called 'tax gain harvesting' — the opposite of tax loss harvesting. Wealthy investors plan their income deliberately to stay under these thresholds. Now you can too."],
    stat="Families under $94K married filing jointly owe ZERO tax on long-term investment gains",action="Check your taxable income this year. If you're near or under the threshold, talk to a tax professional about realizing gains at 0%.",
    questions=[dict(prompt="Who qualifies for the 0% long-term capital gains rate?",opt_a="Only people with no income",opt_b="Only retirees",opt_c="Taxpayers whose taxable income falls below the IRS threshold (~$47K single, ~$94K married)",opt_d="Everyone automatically",correct="C",explanation="The 0% rate applies to taxpayers in the lowest two tax brackets — many working families qualify."),
      dict(prompt="What is 'tax gain harvesting'?",opt_a="Selling investments at a loss",opt_b="Strategically selling appreciated investments in a low-income year to realize gains at 0%",opt_c="Harvesting crops for tax purposes",opt_d="A type of retirement account",correct="B",explanation="Tax gain harvesting resets your cost basis in a tax-free or low-tax year — the opposite of loss harvesting.")]),

  dict(lesson_key="T3",audience="adult",module="Wealth Secrets",order_index=9,icon="👶",color="#22c98a",is_premium=False,
    title="The Kiddie Roth IRA",tagline="Your child's first investment account — started at any age",
    content=["If your child earns ANY income — babysitting, mowing lawns, helping with your business, modeling — they can open a Roth IRA. There is no minimum age requirement.",
      "The rules: They can contribute up to the amount they earned, or the annual Roth IRA limit ($7,000 in 2024), whichever is less. You as the parent can FUND it for them. The money just has to match what they earned.",
      "The compounding math is staggering: $1,000 invested at age 10 at 7% annual return = $29,000+ by age 65. $1,000 invested at age 30 at the same rate = $7,600 by 65. Same money. 20 years difference.",
      "How to document: Keep records of what your child was paid and for what work. A simple spreadsheet works. The IRS needs the income to be real and documentable — not just 'allowance.'"],
    stat="$1,000 in a Roth IRA at age 10 = $29,000+ by retirement at 7% growth",action="Does your child earn any money? Document it this week and open a custodial Roth IRA at Fidelity or Schwab (both free).",
    questions=[dict(prompt="What is the minimum age to open a Roth IRA for a child?",opt_a="18",opt_b="16",opt_c="13",opt_d="There is no minimum age — any earned income qualifies",correct="D",explanation="Any child with documented earned income can have a Roth IRA — parents can fund it on their behalf."),
      dict(prompt="What counts as earned income for a child's Roth IRA?",opt_a="Allowance",opt_b="Birthday money",opt_c="Documented work like babysitting, lawn mowing, or helping a family business",opt_d="Gifts from grandparents",correct="C",explanation="Earned income must be from actual work. Allowance and gifts don't qualify — but real documented work does.")]),

  dict(lesson_key="T4",audience="adult",module="Wealth Secrets",order_index=10,icon="🎓",color="#22c98a",is_premium=False,
    title="529 Plans: Tax-Free Education Money",tagline="Save for education and cut your tax bill at the same time",
    content=["A 529 plan is a tax-advantaged savings account specifically for education expenses. Money goes in after tax, grows completely tax-free, and withdrawals for qualified education expenses are never taxed.",
      "What counts as qualified: K-12 tuition (up to $10,000/year), college tuition and fees, room and board, books, computers, and now — student loan repayment (up to $10,000 lifetime).",
      "New in 2024: Unused 529 funds can be rolled into a Roth IRA for the beneficiary — up to $35,000 lifetime. This eliminates the fear of 'what if my kid doesn't go to college.'",
      "Many states offer a state income tax deduction for contributions to their 529 plan. This can be worth hundreds of dollars per year on top of the tax-free growth."],
    stat="529 invested from birth through age 18 at 7% grows to $100K+ on just $300/month contributions",action="Visit your state's 529 portal or open one at Fidelity/Vanguard. Even $25/month started today makes a meaningful difference.",
    questions=[dict(prompt="What is the primary tax benefit of a 529 plan?",opt_a="Contributions are tax-deductible federally",opt_b="Money grows tax-free and withdrawals for education are not taxed",opt_c="No taxes ever on any withdrawals",opt_d="Employers match contributions",correct="B",explanation="529s offer tax-free growth and tax-free withdrawals for qualified education expenses."),
      dict(prompt="What can you do with unused 529 money as of 2024?",opt_a="Nothing — it's lost",opt_b="Only use it for college",opt_c="Roll it into a Roth IRA for the beneficiary (up to $35K lifetime)",opt_d="Get a full refund",correct="C",explanation="The SECURE 2.0 Act allows up to $35,000 in unused 529 funds to roll into a Roth IRA — eliminating the 'use it or lose it' concern.")]),

  dict(lesson_key="T5",audience="adult",module="Wealth Secrets",order_index=11,icon="💼",color="#7c6af5",is_premium=True,
    title="The Backdoor Roth IRA",tagline="High earners can still access tax-free growth — legally",
    content=["The Roth IRA has income limits — in 2024, single filers above ~$161K and married filers above ~$240K cannot contribute directly. But there is a completely legal workaround called the Backdoor Roth IRA.",
      "How it works: (1) Make a non-deductible contribution to a traditional IRA (anyone can do this regardless of income). (2) Convert that traditional IRA to a Roth IRA shortly after. (3) Report it correctly on Form 8606. That's it.",
      "The key rule: The 'pro-rata rule' means you must be careful if you have other pre-tax IRA money. If you have existing traditional IRAs with pre-tax funds, the conversion becomes partially taxable. Many high earners roll old IRAs into their 401(k) to avoid this.",
      "For married couples, both spouses can each do a Backdoor Roth — doubling the benefit. Many high-earning households use this to put $14,000/year into Roth accounts that will grow tax-free forever."],
    stat="A couple each doing Backdoor Roth for 20 years = $280,000+ in tax-free retirement wealth",action="If you earn above Roth income limits, consult a CPA about executing a Backdoor Roth this year. The deadline is tax day.",
    questions=[dict(prompt="Who can use the Backdoor Roth IRA strategy?",opt_a="Only people under the income limit",opt_b="High earners who exceed direct Roth IRA income limits",opt_c="Only retirees",opt_d="Only people with no existing IRAs",correct="B",explanation="The Backdoor Roth is specifically designed for high earners who cannot contribute directly to a Roth IRA."),
      dict(prompt="What is the 'pro-rata rule' in Backdoor Roth conversions?",opt_a="You must wait one year before converting",opt_b="Existing pre-tax IRA money affects the tax calculation of the conversion",opt_c="Only one conversion per year allowed",opt_d="The IRS takes a 10% fee",correct="B",explanation="If you have existing pre-tax IRA money, the IRS requires you to calculate the taxable portion proportionally — this can create unexpected taxes.")]),

  dict(lesson_key="T6",audience="adult",module="Wealth Secrets",order_index=12,icon="📉",color="#7c6af5",is_premium=True,
    title="Tax Loss Harvesting",tagline="Turn investment losses into tax savings — legally",
    content=["Tax loss harvesting is the strategy of selling investments that have declined in value to realize a loss on paper — which then offsets capital gains and reduces your tax bill.",
      "How it works: If you have $10,000 in capital gains from selling one investment, but also have another investment sitting at a $4,000 loss, you can sell the loser, capture the $4,000 loss, and only pay tax on $6,000 net gains.",
      "The wash-sale rule: You cannot buy back the same or 'substantially identical' security within 30 days before or after the sale. But you CAN buy a similar fund immediately — sell a S&P 500 fund and buy a Total Market fund the same day.",
      "Up to $3,000 of net capital losses per year can be deducted against ordinary income. Losses beyond $3,000 carry forward to future years indefinitely. Wealthy investors do this systematically every year — and now you can too."],
    stat="Systematic tax loss harvesting can save $1,500–$5,000+ per year for active investors",action="Review your taxable investment accounts at year end. Identify any positions sitting at a loss and evaluate whether harvesting makes sense.",
    questions=[dict(prompt="What is the primary goal of tax loss harvesting?",opt_a="To eliminate all investments",opt_b="To sell your worst investments permanently",opt_c="To realize investment losses that offset capital gains and reduce taxes",opt_d="To avoid paying any taxes ever",correct="C",explanation="Tax loss harvesting converts paper losses into real tax savings by offsetting gains elsewhere in your portfolio."),
      dict(prompt="What is the wash-sale rule?",opt_a="You must wash your hands before trading",opt_b="You cannot repurchase the same or substantially identical security within 30 days of the sale",opt_c="Losses can only be claimed once",opt_d="You must wait one year to sell",correct="B",explanation="The wash-sale rule prevents you from immediately buying back the same investment — but you can buy a similar (not identical) fund right away.")]),

  dict(lesson_key="T7",audience="adult",module="Wealth Secrets",order_index=13,icon="🏠",color="#7c6af5",is_premium=True,
    title="The Augusta Rule",tagline="Rent your home to your business — 14 days completely tax-free",
    content=["Section 280A of the tax code — nicknamed the 'Augusta Rule' after Augusta, Georgia homeowners who rent their homes during The Masters golf tournament — allows you to rent your personal home for up to 14 days per year completely tax-free.",
      "How business owners use it: If you own a business (even a side hustle LLC), your business can rent your home for meetings, retreats, or training sessions. You receive the rental income personally — completely tax-free. The business deducts it as a business expense.",
      "Example: Your business pays you $3,000 to use your home for a 2-day business retreat. The business deducts $3,000 (reduces business taxes). You receive $3,000 personally — not reported as income, not taxed. Net benefit: $3,000 in your pocket at near-zero tax.",
      "Requirements: The rental must be at fair market value (get comparable rates from hotels or event venues in your area). Document everything — meeting agenda, attendees, purpose. This must be a legitimate business activity."],
    stat="Business owners can legally transfer $5,000–$15,000/year from business to personal tax-free using this strategy",action="If you have a business entity, research the Augusta Rule with your CPA. Keep documentation of any home business meetings.",
    questions=[dict(prompt="How many days can you rent your home to your business tax-free under the Augusta Rule?",opt_a="7 days",opt_b="14 days",opt_c="30 days",opt_d="Unlimited",correct="B",explanation="Section 280A allows up to 14 days of tax-free rental income from your personal residence per year."),
      dict(prompt="What is required for the Augusta Rule to be valid?",opt_a="You must live in Augusta, Georgia",opt_b="The rental must be at fair market rates with documented legitimate business purpose",opt_c="Your business must be a corporation",opt_d="You must own your home outright",correct="B",explanation="The rental must be at fair market value and serve a genuine business purpose — documentation is essential.")]),

  dict(lesson_key="T8",audience="adult",module="Wealth Secrets",order_index=14,icon="🚀",color="#7c6af5",is_premium=True,
    title="The Solo 401(k): $69,000/Year Tax-Deferred",tagline="Self-employed? You have access to the most powerful retirement account",
    content=["If you have any self-employment income — freelance work, a side business, 1099 income — you qualify for a Solo 401(k), also called an Individual 401(k). The contribution limits are dramatically higher than a regular 401(k).",
      "2024 limits: As the 'employee,' you can contribute up to $23,000 (same as regular 401k). As the 'employer' (your own business), you can contribute an additional 25% of net self-employment income. Total limit: $69,000 per year.",
      "Roth Solo 401(k) option: Many Solo 401(k) plans now offer a Roth option — contribute after-tax and let it grow tax-free forever. This is a powerful combination for high-earning self-employed individuals.",
      "Who qualifies: Any self-employed person with no full-time employees (a spouse can also participate). This includes freelancers, consultants, Etsy sellers, Uber drivers, and anyone with Schedule C income. You do NOT need it to be your primary income."],
    stat="A self-employed person maximizing a Solo 401(k) at $69,000/year saves $15,000–$25,000+ in taxes annually",action="If you have any 1099 or self-employment income, open a Solo 401(k) at Fidelity or Schwab (both free). Do this before December 31.",
    questions=[dict(prompt="Who qualifies for a Solo 401(k)?",opt_a="Only full-time business owners",opt_b="Only people with no other income",opt_c="Any person with self-employment income and no full-time employees",opt_d="Only incorporated businesses",correct="C",explanation="Any self-employed person — including freelancers and side hustlers — with no full-time employees can open a Solo 401(k)."),
      dict(prompt="What is the maximum Solo 401(k) contribution in 2024?",opt_a="$7,000",opt_b="$23,000",opt_c="$46,000",opt_d="$69,000 (employee + employer contributions combined)",correct="D",explanation="The Solo 401(k) allows up to $69,000 in total contributions — employee contributions plus 25% employer contribution from business profits.")]),

  dict(lesson_key="T9",audience="adult",module="Wealth Secrets",order_index=15,icon="🎁",color="#7c6af5",is_premium=True,
    title="The $18,000 Annual Gift Strategy",tagline="Transfer wealth to your children every year — completely tax-free",
    content=["The IRS allows you to give up to $18,000 per year to any individual — completely free of gift tax and without filing any paperwork. This is called the annual gift tax exclusion.",
      "How wealthy families use it: Two parents can each give $18,000 to each child — that's $36,000 per child per year, completely tax-free. With multiple children and grandchildren, families legally transfer hundreds of thousands per year without any gift or estate tax consequences.",
      "What you can do with it: Give cash, securities, or fund a 529 plan. You can even 'superfund' a 529 — contribute 5 years of gifts at once ($90,000 per person, $180,000 per couple) and it counts as if spread over 5 years for gift tax purposes.",
      "This is one of the primary wealth transfer strategies used by affluent families. It removes assets from your taxable estate while alive, reducing potential estate taxes and transferring compound growth to the next generation."],
    stat="Two parents gifting $36,000/year to two children for 20 years = $1.4M+ transferred tax-free",action="If reducing your estate or helping your children build wealth is a goal, start annual gifting this year. No forms required for amounts under $18,000 per recipient.",
    questions=[dict(prompt="How much can each person give tax-free to any individual per year in 2024?",opt_a="$5,000",opt_b="$10,000",opt_c="$18,000",opt_d="$50,000",correct="C",explanation="The 2024 annual gift tax exclusion is $18,000 per donor per recipient — no gift tax, no forms required."),
      dict(prompt="What is '529 superfunding'?",opt_a="Investing aggressively in a 529",opt_b="Contributing 5 years of gift tax exclusions at once to a 529 plan",opt_c="A type of 529 plan for athletes",opt_d="Matching contributions from the government",correct="B",explanation="Superfunding allows you to contribute up to 5 years of annual gift exclusions ($90,000 per person) to a 529 in one year, treated as if spread over 5 years.")]),
]

def seed_edu_lessons():
    if EduLesson.query.count(): return
    for ld in EDU_SEED:
        questions = ld.pop("questions"); content = ld.pop("content")
        lesson = EduLesson(content_json=json.dumps(content), **ld)
        db.session.add(lesson); db.session.flush()
        for q in questions:
            db.session.add(EduQuizQuestion(lesson_id=lesson.id, **q))
    db.session.commit()

# ── Helpers ───────────────────────────────────────────────────────────────────

def me() -> Optional[User]:
    uid = session.get("uid")
    return db.session.get(User, uid) if uid else None

def require_login():
    if not me(): flash("Login required.", "warning"); return redirect(url_for("login"))
    return None

def ensure_profile(u):
    p = Profile.query.filter_by(user_id=u.id).first()
    if p: return p
    p = Profile(user_id=u.id, display_name="Household")
    db.session.add(p); db.session.commit(); return p

def money(x): return f"${x:,.2f}"
def safe(s):  return (s or "").replace("&","&amp;").replace("<","&lt;").replace(">","&gt;")
def is_org_admin(u): return bool(u.org_id) and (u.org_role == "admin")
def premium_gate(u): return bool(u.is_premium) or is_org_admin(u) or bool(u.org_id)

CATEGORY_RULES = [
    ("Rent/Mortgage", ["rent","mortgage"]),
    ("Utilities",     ["electric","water","gas","utility","comcast","internet","phone"]),
    ("Groceries",     ["kroger","heb","walmart","target","grocery","whole foods","aldi"]),
    ("Restaurants",   ["mcdonald","chick","starbucks","restaurant","doordash","ubereats"]),
    ("Transport",     ["shell","exxon","chevron","gas station","uber","lyft"]),
    ("Subscriptions", ["netflix","spotify","hulu","prime","apple","google","subscription"]),
    ("Debt",          ["credit card","loan","affirm","klarna"]),
    ("Income",        ["payroll","paycheck","direct deposit","salary"]),
    ("Investing",     ["vanguard","fidelity","schwab","robinhood","brokerage"]),
]

def categorize(description, amount):
    d = (description or "").lower()
    if amount > 0:
        for cat, keys in CATEGORY_RULES:
            if cat == "Income" and any(k in d for k in keys): return "Income"
        return "Income"
    for cat, keys in CATEGORY_RULES:
        if any(k in d for k in keys): return cat
    return "Other"

def compute_freedom_score(u):
    p = ensure_profile(u)
    net = p.monthly_income - (p.fixed_bills + p.variable_spend + p.debt_minimums)
    net_score  = max(0.0, min(1.0, net / max(1.0, p.monthly_income) / 0.20)) * 25.0 if p.monthly_income > 0 else 0.0
    ef_ratio   = min(1.0, p.emergency_fund_current / p.emergency_fund_target) if p.emergency_fund_target > 0 else 0.0
    ef_score   = ef_ratio * 20.0
    debt_ratio = p.debt_minimums / p.monthly_income if p.monthly_income > 0 else 1.0
    debt_score = (1.0 - min(1.0, debt_ratio / 0.25)) * 20.0
    inv_ratio  = min(1.0, (p.monthly_investing_target / p.monthly_income) / 0.10) if p.monthly_income > 0 and p.monthly_investing_target > 0 else 0.0
    inv_score  = inv_ratio * 15.0
    since = dt.date.today() - dt.timedelta(days=30)
    txs   = Transaction.query.filter(Transaction.user_id == u.id, Transaction.date >= since).all()
    spend = sum(-t.amount for t in txs if t.amount < 0)
    leak  = sum(-t.amount for t in txs if t.amount < 0 and t.category in ("Subscriptions","Restaurants"))
    leak_ratio = 0.0 if spend <= 0 else leak / spend
    leak_score = (1.0 - min(1.0, max(0.0, (leak_ratio - 0.10) / 0.20))) * 20.0
    total = max(0, min(100, int(round(net_score + ef_score + debt_score + inv_score + leak_score))))
    return total, {"net_points":net_score,"ef_points":ef_score,"debt_points":debt_score,
                   "inv_points":inv_score,"leak_points":leak_score,
                   "net":net,"leak_ratio":leak_ratio,"debt_ratio":debt_ratio,"ef_ratio":ef_ratio}

def generate_alerts(u):
    p = ensure_profile(u); score, parts = compute_freedom_score(u); alerts = []
    if p.monthly_income > 0:
        if parts["net"] < 0: alerts.append("⚠ Cashflow is negative — spending more than you make.")
        elif parts["net"] < 0.05 * p.monthly_income: alerts.append("⚠ Cashflow is tight (<5% net).")
    if p.emergency_fund_target > 0 and p.emergency_fund_current < min(500.0, p.emergency_fund_target):
        alerts.append("⚠ Emergency fund under $500. Build a buffer.")
    if parts["debt_ratio"] >= 0.20: alerts.append("⚠ Debt minimums are heavy (≥20% of income).")
    if p.monthly_income > 0 and (p.monthly_investing_target / max(1, p.monthly_income)) < 0.03:
        alerts.append("⚠ Investing target is low (<3% of income).")
    if parts["leak_ratio"] >= 0.25: alerts.append("⚠ Subscriptions + dining eating ≥25% of spend.")
    if score >= 80: alerts.append("✓ Strong position. Next: raise investing rate.")
    elif score >= 60: alerts.append("✓ Stable. Push emergency fund + automate investing.")
    else: alerts.append("→ Recovery mode. Cut leaks, stabilize cashflow, then invest.")
    return alerts[:6]

@dataclass
class DebtRow:
    name: str; balance: float; apr: float; minpay: float

def payoff_schedule(debts, extra_per_month, method, max_months=240):
    if not debts: return 0, 0.0, []
    items = [DebtRow(d.name, float(d.balance), float(d.apr), float(d.minimum_payment)) for d in debts]
    months = 0; total_interest = 0.0; timeline = []
    def sort_key(d): return (d.balance, -d.apr) if method == "snowball" else (-d.apr, d.balance)
    while months < max_months and any(d.balance > 0.01 for d in items):
        months += 1; month_interest = 0.0
        for d in items:
            if d.balance <= 0: continue
            interest = d.balance * ((d.apr / 100.0) / 12.0)
            d.balance += interest; month_interest += interest
        total_interest += month_interest; available_extra = max(0.0, float(extra_per_month)); paid = []
        for d in items:
            if d.balance <= 0: continue
            pay = min(d.balance, d.minpay); d.balance -= pay; paid.append((d.name, pay))
        for d in sorted([x for x in items if x.balance > 0.01], key=sort_key):
            if available_extra <= 0: break
            pay = min(d.balance, available_extra); d.balance -= pay; available_extra -= pay; paid.append((d.name, pay))
        timeline.append({"month":months,"interest":month_interest,"payments":paid,"remaining":sum(max(0.0,d.balance) for d in items)})
    return months, total_interest, timeline

def future_value(monthly, years, annual_return):
    r = (annual_return / 100.0) / 12.0; n = years * 12
    return monthly * n if r == 0 else monthly * ((pow(1+r,n)-1)/r)

def retirement_projection(current_age, retire_age, monthly_invest, annual_return):
    years = max(0, retire_age - current_age); fv = future_value(monthly_invest, years, annual_return)
    return {"years":years,"fv":fv,"annual_income":fv*0.04,"monthly_income":(fv*0.04)/12}

# ── Base Template ─────────────────────────────────────────────────────────────

BASE = r"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
  <title>{{ title }} | LegacyLift</title>
  <link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,600;0,700;1,400&family=DM+Sans:wght@300;400;500;600&family=DM+Mono:wght@400;500&display=swap" rel="stylesheet">
  <style>
    :root{
      --bg:#0B1724;--surface:#111827;--surface2:#1e293b;--surface3:#243044;
      --border:rgba(212,175,55,.15);--border2:rgba(212,175,55,.28);--border3:rgba(255,255,255,.08);
      --gold:#D4AF37;--gold2:#e8c84a;--lift:#7c6af5;--lift2:#a89cf8;
      --green:#16A34A;--green2:#22c98a;--amber:#e8a030;--red:#f05555;--blue:#1E3A8A;
      --text:#f8fafc;--muted:#94a3b8;--muted2:#475569;
      --serif:'Cormorant Garamond',Georgia,serif;--sans:'DM Sans',sans-serif;--mono:'DM Mono',monospace;
    }
    *,*::before,*::after{box-sizing:border-box;margin:0;padding:0;}
    body{background:var(--bg);color:var(--text);font-family:var(--sans);font-size:15px;line-height:1.65;min-height:100vh;-webkit-font-smoothing:antialiased;}
    nav{background:var(--surface);border-bottom:1px solid var(--border3);padding:0 1.75rem;height:56px;display:flex;align-items:center;justify-content:space-between;position:sticky;top:0;z-index:100;}
    .nav-brand-wrap{display:flex;align-items:center;gap:.55rem;text-decoration:none;}
    .nav-brand-ic{width:28px;height:28px;border-radius:7px;background:linear-gradient(135deg,#1E3A8A,#D4AF37);display:flex;align-items:center;justify-content:center;font-family:var(--mono);font-size:.72rem;font-weight:700;color:#fff;flex-shrink:0;}
    .nav-brand-txt{font-family:var(--serif);font-size:1.15rem;font-weight:600;background:linear-gradient(135deg,var(--lift2),var(--gold));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;}
    nav ul{list-style:none;display:flex;gap:.15rem;align-items:center;flex-wrap:wrap;}
    nav a{color:var(--muted);text-decoration:none;font-family:var(--sans);font-size:.8rem;font-weight:500;padding:.38rem .8rem;border:1px solid transparent;border-radius:7px;transition:all .18s;}
    nav a:hover{color:var(--text);background:rgba(255,255,255,.05);}
    nav a.active{color:var(--lift2);background:rgba(124,106,245,.1);border-color:rgba(124,106,245,.2);}
    .btn-nav-cta{background:var(--gold)!important;color:#0B1724!important;font-weight:700!important;border:none!important;}
    main{max-width:1200px;margin:0 auto;padding:2rem 1.75rem;}
    .card{background:var(--surface);border:1px solid var(--border3);border-radius:12px;padding:1.35rem;transition:border-color .2s;box-shadow:0 2px 8px rgba(0,0,0,.2);}
    .card:hover{border-color:var(--border2);}.card-sm{padding:1rem 1.15rem;}
    .card-gold{background:linear-gradient(135deg,rgba(200,168,75,.09),rgba(200,168,75,.03));border-color:var(--border2);}
    .card-lift{background:linear-gradient(135deg,rgba(124,106,245,.09),rgba(124,106,245,.03));border-color:rgba(124,106,245,.2);}
    .grid{display:grid;gap:1rem;}.grid-2{grid-template-columns:1fr 1fr;}.grid-3{grid-template-columns:repeat(3,1fr);}.grid-4{grid-template-columns:repeat(4,1fr);}
    @media(max-width:768px){.grid-2,.grid-3,.grid-4{grid-template-columns:1fr;}}
    h1,h2,h3,h4{font-family:var(--serif);font-weight:600;color:var(--text);}
    h1{font-size:1.8rem;margin-bottom:.5rem;}h2{font-size:1.35rem;margin-bottom:.5rem;}
    .mono{font-family:var(--mono);}.muted{color:var(--muted);}.green{color:var(--green2);}.amber{color:var(--amber);}.red{color:var(--red);}.gold{color:var(--gold2);}.lift{color:var(--lift2);}
    .bar-wrap{background:rgba(255,255,255,.05);border-radius:3px;height:5px;overflow:hidden;}
    .bar{height:5px;border-radius:3px;transition:width .5s;}.bar-green{background:var(--green);}.bar-amber{background:var(--amber);}.bar-red{background:var(--red);}
    .stat-box{background:var(--surface2);border:1px solid var(--border3);border-radius:10px;padding:.95rem;box-shadow:0 2px 6px rgba(0,0,0,.15);}
    .stat-box .val{font-family:var(--mono);font-size:1.4rem;font-weight:500;color:var(--text);margin-bottom:.15rem;}
    .stat-box .lbl{font-size:.65rem;color:var(--muted);letter-spacing:.08em;text-transform:uppercase;}
    .alert-item{display:flex;align-items:flex-start;gap:.55rem;padding:.65rem .9rem;border-radius:8px;margin-bottom:.45rem;font-size:.78rem;line-height:1.5;border:1px solid transparent;}
    .alert-item.warn{background:rgba(240,85,85,.07);border-color:rgba(240,85,85,.18);color:#f08888;}
    .alert-item.ok{background:rgba(34,201,138,.07);border-color:rgba(34,201,138,.18);color:var(--green2);}
    .alert-item:not(.warn):not(.ok){background:rgba(255,255,255,.04);border-color:var(--border3);color:var(--muted);}
    label{display:block;font-size:.65rem;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);margin-bottom:.28rem;}
    input[type=text],input[type=email],input[type=password],input[type=number],input[type=file],select,textarea{width:100%;background:var(--surface3);border:1px solid var(--border3);color:var(--text);font-family:var(--sans);font-size:.85rem;padding:.52rem .8rem;border-radius:8px;outline:none;transition:border-color .18s;}
    input:focus,select:focus,textarea:focus{border-color:rgba(124,106,245,.5);}
    .form-group{margin-bottom:1rem;}
    .btn{display:inline-block;font-family:var(--sans);font-size:.8rem;font-weight:600;padding:.52rem 1.15rem;border-radius:8px;border:1px solid var(--border2);cursor:pointer;text-decoration:none;color:var(--text);background:rgba(255,255,255,.05);transition:all .18s;white-space:nowrap;}
    .btn:hover{background:rgba(255,255,255,.09);border-color:rgba(200,168,75,.35);}
    .btn-primary{background:var(--gold);color:#0B1724;border:none;box-shadow:0 4px 14px rgba(212,175,55,.3);font-weight:700;}
    .btn-primary:hover{background:var(--gold2);box-shadow:0 6px 20px rgba(212,175,55,.45);transform:translateY(-1px);}
    .btn-gold{background:linear-gradient(135deg,var(--gold),#a8892a);color:var(--bg);border:none;}
    .btn-danger{background:transparent;border-color:var(--red);color:var(--red);}
    .btn-amber{border-color:var(--amber);color:var(--amber);}
    .btn-sm{padding:.32rem .75rem;font-size:.73rem;}.btn-block{display:block;width:100%;text-align:center;}
    table{width:100%;border-collapse:collapse;font-size:.8rem;}
    th{font-family:var(--mono);font-size:.62rem;letter-spacing:.09em;text-transform:uppercase;color:var(--muted);border-bottom:1px solid var(--border3);padding:.45rem .65rem;text-align:left;}
    td{padding:.52rem .65rem;border-bottom:1px solid rgba(255,255,255,.03);font-family:var(--mono);font-size:.76rem;}
    tr:last-child td{border-bottom:none;}tr:hover td{background:var(--surface2);}
    .text-right{text-align:right;}
    .flash-list{list-style:none;margin-bottom:1rem;}
    .flash-item{padding:.65rem 1rem;border-left:3px solid var(--muted);margin-bottom:.4rem;font-size:.82rem;background:var(--surface2);border-radius:0 8px 8px 0;}
    .flash-item.success{border-color:var(--green);color:var(--green2);}.flash-item.warning{border-color:var(--amber);color:var(--amber);}.flash-item.error{border-color:var(--red);color:var(--red);}
    hr{border:none;border-top:1px solid var(--border3);margin:1.25rem 0;}
    .badge{display:inline-flex;align-items:center;font-family:var(--mono);font-size:.6rem;font-weight:700;padding:.15rem .48rem;border-radius:4px;letter-spacing:.06em;text-transform:uppercase;}
    .badge-green{background:rgba(34,201,138,.12);color:var(--green2);border:1px solid rgba(34,201,138,.22);}
    .badge-amber,.badge-gold{background:rgba(200,168,75,.14);color:var(--gold2);border:1px solid rgba(200,168,75,.28);}
    .badge-lift{background:rgba(124,106,245,.14);color:var(--lift2);border:1px solid rgba(124,106,245,.28);}
    .badge-pink{background:rgba(240,85,85,.12);color:#f08888;border:1px solid rgba(240,85,85,.25);}
    .badge-muted{background:rgba(255,255,255,.06);color:var(--muted);border:1px solid var(--border3);}
    .section-label{font-family:var(--mono);font-size:.6rem;letter-spacing:.12em;text-transform:uppercase;color:var(--gold2);background:rgba(212,175,55,.1);padding:.18rem .52rem;border-radius:4px;border:1px solid rgba(212,175,55,.2);display:inline-block;margin-bottom:.65rem;}
    footer{background:#0B1724;border-top:1px solid rgba(212,175,55,.15);padding:1.1rem 1.75rem;font-size:.76rem;color:var(--muted);text-align:center;margin-top:2rem;}
    footer a{color:var(--muted);text-decoration:none;}footer a:hover{color:var(--gold2);}
    @keyframes fadeUp{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}
    .fade-in{animation:fadeUp .32s ease both;}
    input[type=range]{accent-color:var(--lift);cursor:pointer;width:100%;}
    input[type=radio]{accent-color:var(--lift);width:auto;}
    input[type=checkbox]{accent-color:var(--green);cursor:pointer;}
    ::-webkit-scrollbar{width:4px;}::-webkit-scrollbar-thumb{background:var(--border2);border-radius:2px;}
  </style>
</head>
<body>
<nav>
  <a class="nav-brand-wrap" href="{{ url_for('home') }}">
    <div class="nav-brand-ic">LL</div>
    <span class="nav-brand-txt">LegacyLift</span>
  </a>
  <ul>
    {% if u %}
      <li><a href="{{ url_for('dashboard') }}">Dashboard</a></li>
      <li><a href="{{ url_for('scoreboard') }}">Score</a></li>
      <li><a href="{{ url_for('edu_learn') }}" class="{{ 'active' if active_page=='learn' else '' }}">Education</a></li>
      <li><a href="{{ url_for('budget') }}">Budget</a></li>
      <li><a href="{{ url_for('transactions') }}">Cashflow</a></li>
      <li><a href="{{ url_for('debt') }}">Debt</a></li>
      <li><a href="{{ url_for('retirement') }}">Retirement</a></li>
      <li><a href="{{ url_for('community') }}">Community</a></li>
      <li><a href="{{ url_for('premium') }}">
        {% if u.is_premium or u.org_id %}<span class="green">PRO</span>{% else %}Upgrade{% endif %}
      </a></li>
      {% if u.org_role == 'admin' %}<li><a href="{{ url_for('admin_lessons') }}">Admin</a></li>{% endif %}
      <li><a href="{{ url_for('logout') }}">Sign Out</a></li>
    {% else %}
      <li><a href="{{ url_for('login') }}">Login</a></li>
      <li><a href="{{ url_for('signup') }}" class="btn-nav-cta btn">Start Free</a></li>
    {% endif %}
  </ul>
</nav>
<main class="fade-in">
  {% with messages = get_flashed_messages(with_categories=true) %}
    {% if messages %}<ul class="flash-list">
      {% for cat, msg in messages %}<li class="flash-item {{ cat }}">{{ msg }}</li>{% endfor %}
    </ul>{% endif %}
  {% endwith %}
  {{ body|safe }}
</main>
<footer>
  &copy; {{ year }} LegacyLift LLC &nbsp;|&nbsp; Education only - not financial advice &nbsp;|&nbsp;
  <a href="/privacy">Privacy</a> &nbsp;.&nbsp; <a href="/terms">Terms</a> &nbsp;.&nbsp;
  <a href="/disclaimer">Disclaimer</a> &nbsp;.&nbsp; <a href="/about">Our Story</a> &nbsp;.&nbsp; <a href="/support">Support</a>
</footer>
</body>
</html>"""


def page(title, body, active_page=""):
    u = me()
    return render_template_string(BASE, title=title, body=body, u=u,
                                  year=dt.datetime.utcnow().year, active_page=active_page)

# ── Auth ──────────────────────────────────────────────────────────────────────

@app.get("/")
def home():
    u = me()
    if u: return redirect(url_for("dashboard"))
    body = """
<style>
@import url('https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,600;0,700;1,400&family=DM+Sans:wght@300;400;500;600&family=DM+Mono:wght@400;500&display=swap');
*{box-sizing:border-box;}
.ll-page{margin:-1rem;padding:0;font-family:'DM Sans',sans-serif;}

/* NAV */
.ll-nav{display:flex;align-items:center;justify-content:space-between;padding:1.2rem 3rem;
  background:#0B1724;backdrop-filter:blur(10px);border-bottom:1px solid rgba(255,255,255,.08);
  position:sticky;top:0;z-index:100;}
.ll-brand-wrap{display:flex;align-items:center;gap:.6rem;text-decoration:none;}
.ll-brand-ic{width:32px;height:32px;border-radius:8px;background:linear-gradient(135deg,#7c6af5,#D4AF37);
  display:flex;align-items:center;justify-content:center;font-weight:700;font-size:.8rem;color:#fff;font-family:monospace;}
.ll-brand-txt{font-family:'Cormorant Garamond',serif;font-size:1.25rem;font-weight:600;
  background:linear-gradient(135deg,#a89cf8,#D4AF37);-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;}
.ll-nav-r{display:flex;align-items:center;gap:.5rem;}
.ll-nbtn{background:transparent;border:1px solid rgba(255,255,255,.15);color:#cbd5e1;padding:.4rem .9rem;
  border-radius:8px;text-decoration:none;font-size:.8rem;transition:all .18s;}
.ll-nbtn:hover{background:rgba(255,255,255,.08);color:#fff;}
.ll-nbtn-cta{background:#D4AF37;color:#0B1724;padding:.45rem 1.25rem;border-radius:8px;
  text-decoration:none;font-size:.82rem;font-weight:700;transition:all .2s;box-shadow:0 4px 14px rgba(212,175,55,.3);}
.ll-nbtn-cta:hover{background:#e8c84a;transform:translateY(-1px);}

/* HERO — dark navy */
.ll-hero{background:linear-gradient(160deg,#0B1724 0%,#111827 60%,#0B1724 100%);
  padding:5rem 2rem 4.5rem;text-align:center;position:relative;overflow:hidden;}
.ll-hero::before{content:'';position:absolute;inset:0;pointer-events:none;
  background:radial-gradient(ellipse 70% 50% at 50% 0%,rgba(212,175,55,.07),transparent 65%);}
.ll-pill{display:inline-block;font-size:.63rem;letter-spacing:.16em;text-transform:uppercase;
  color:#a89cf8;border:1px solid rgba(124,106,245,.3);padding:.28rem .9rem;border-radius:20px;
  margin-bottom:1.25rem;background:rgba(124,106,245,.08);}
.ll-h1{font-family:'Cormorant Garamond',serif;font-size:clamp(2.4rem,5.5vw,4.2rem);
  color:#f8fafc;line-height:1.08;margin:0 auto 1rem;max-width:780px;}
.ll-gold{color:#D4AF37;}
.ll-lift{color:#a89cf8;}
.ll-sub{font-size:1rem;color:#94a3b8;max-width:600px;margin:0 auto 1rem;line-height:1.8;}
.ll-sub2{font-size:.9rem;color:#64748b;max-width:580px;margin:0 auto 2rem;line-height:1.75;}
.ll-btns{display:flex;gap:.85rem;justify-content:center;flex-wrap:wrap;margin-bottom:1.75rem;}
.ll-btn-gold{background:#D4AF37;color:#0B1724;padding:.88rem 2.3rem;border-radius:10px;
  text-decoration:none;font-weight:700;font-size:.95rem;transition:all .2s;
  box-shadow:0 4px 18px rgba(212,175,55,.35);}
.ll-btn-gold:hover{background:#e8c84a;box-shadow:0 8px 28px rgba(212,175,55,.5);transform:translateY(-2px);}
.ll-btn-ghost{background:transparent;color:#f8fafc;padding:.88rem 2rem;border-radius:10px;
  text-decoration:none;font-weight:600;font-size:.95rem;border:1px solid rgba(255,255,255,.2);transition:all .18s;}
.ll-btn-ghost:hover{background:rgba(255,255,255,.07);border-color:rgba(212,175,55,.4);}
.ll-trust{display:flex;gap:1.5rem;justify-content:center;flex-wrap:wrap;font-size:.73rem;color:#475569;}
.ll-trust span::before{content:'✦ ';color:#D4AF37;font-size:.58rem;}

/* STRIP — dark */
.ll-strip{background:#111827;border-top:1px solid rgba(255,255,255,.06);border-bottom:1px solid rgba(255,255,255,.06);
  padding:.85rem 2rem;display:flex;justify-content:center;gap:2.5rem;flex-wrap:wrap;}
.ll-si{font-size:.72rem;color:#64748b;display:flex;gap:.4rem;align-items:center;}
.ll-si strong{color:#D4AF37;font-weight:600;}

/* HEADLINE SECTION — light */
.ll-headline{background:#F4F6F8;padding:4rem 2rem;text-align:center;}
.ll-hl-title{font-family:'Cormorant Garamond',serif;font-size:clamp(1.8rem,4vw,2.8rem);
  font-weight:600;color:#111827;max-width:720px;margin:0 auto .85rem;line-height:1.2;}
.ll-hl-sub{font-size:.95rem;color:#374151;max-width:620px;margin:0 auto 1.75rem;line-height:1.8;}
.ll-hl-cta{display:inline-block;background:#D4AF37;color:#0B1724;padding:.82rem 2.2rem;
  border-radius:10px;font-weight:700;font-size:.95rem;text-decoration:none;
  box-shadow:0 4px 16px rgba(212,175,55,.35);transition:all .2s;}
.ll-hl-cta:hover{background:#e8c84a;transform:translateY(-2px);}

/* WHAT YOU GET — light gray */
.ll-wyg{background:#F8FAFC;padding:5rem 2rem;}
.ll-wyg-inner{max-width:1100px;margin:0 auto;}
.ll-wyg-tag{font-size:.62rem;letter-spacing:.14em;text-transform:uppercase;color:#1E3A8A;
  background:rgba(30,58,138,.08);padding:.2rem .65rem;border-radius:4px;
  border:1px solid rgba(30,58,138,.15);display:inline-block;margin-bottom:.65rem;}
.ll-wyg-title{font-family:'Cormorant Garamond',serif;font-size:2.2rem;font-weight:600;
  color:#111827;margin-bottom:.4rem;}
.ll-wyg-sub{font-size:.88rem;color:#6b7280;margin-bottom:2.5rem;}
.ll-wyg-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(240px,1fr));gap:1.1rem;}
.ll-wyg-card{background:#fff;border:1px solid #e5e7eb;border-radius:14px;padding:1.5rem;
  box-shadow:0 2px 8px rgba(0,0,0,.06);transition:all .22s;}
.ll-wyg-card:hover{border-color:#D4AF37;transform:translateY(-3px);box-shadow:0 8px 24px rgba(0,0,0,.1);}
.ll-wyg-ic{font-size:1.75rem;margin-bottom:.85rem;}
.ll-wyg-ct{font-family:'Cormorant Garamond',serif;font-size:1.05rem;font-weight:600;
  color:#111827;margin-bottom:.35rem;}
.ll-wyg-cd{font-size:.8rem;color:#6b7280;line-height:1.65;}

/* TOOLS SECTION — white */
.ll-tools{background:#fff;padding:5rem 2rem;}
.ll-tools-inner{max-width:1100px;margin:0 auto;}
.ll-tools-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(275px,1fr));gap:1rem;}
.ll-tool-card{background:#F8FAFC;border:1px solid #e5e7eb;border-radius:14px;padding:1.5rem;
  text-decoration:none;display:block;transition:all .22s;}
.ll-tool-card:hover{border-color:#1E3A8A;transform:translateY(-3px);box-shadow:0 8px 24px rgba(0,0,0,.08);}
.ll-tool-ic{font-size:1.75rem;margin-bottom:.85rem;}
.ll-tool-t{font-family:'Cormorant Garamond',serif;font-size:1.05rem;font-weight:600;
  color:#111827;margin-bottom:.35rem;}
.ll-tool-d{font-size:.8rem;color:#6b7280;line-height:1.65;}

/* INSIDE LEGACYLIFT — light */
.ll-inside{background:#F4F6F8;padding:5rem 2rem;}
.ll-inside-inner{max-width:1100px;margin:0 auto;}
.ll-inside-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(200px,1fr));gap:.85rem;margin-top:2.5rem;}
.ll-inside-card{background:#fff;border:1px solid #e5e7eb;border-radius:10px;padding:1.1rem;
  text-align:center;box-shadow:0 2px 6px rgba(0,0,0,.05);}
.ll-inside-ic{font-size:1.5rem;margin-bottom:.5rem;}
.ll-inside-t{font-size:.82rem;font-weight:600;color:#111827;}
.ll-inside-d{font-size:.72rem;color:#6b7280;margin-top:.2rem;line-height:1.5;}

/* WHY I BUILT IT — dark */
.ll-why{background:#0B1724;padding:5rem 2rem;text-align:center;}
.ll-why-inner{max-width:720px;margin:0 auto;}
.ll-why-tag{font-size:.62rem;letter-spacing:.14em;text-transform:uppercase;color:#D4AF37;
  margin-bottom:.75rem;display:block;}
.ll-why-title{font-family:'Cormorant Garamond',serif;font-size:2rem;font-weight:600;
  color:#f8fafc;margin-bottom:1.25rem;}
.ll-why-body{font-size:.92rem;color:#94a3b8;line-height:1.85;margin-bottom:1.5rem;}

/* TESTIMONIALS — dark */
.ll-tests{background:#111827;padding:4rem 2rem;}
.ll-tgrid{display:grid;grid-template-columns:repeat(auto-fill,minmax(300px,1fr));gap:1rem;max-width:1100px;margin:0 auto;}
.ll-tc{background:#1e293b;border:1px solid rgba(255,255,255,.07);border-radius:12px;padding:1.4rem;}
.ll-ts{color:#D4AF37;font-size:.88rem;letter-spacing:.08em;margin-bottom:.7rem;}
.ll-tb{font-size:.84rem;color:#94a3b8;line-height:1.7;font-style:italic;margin-bottom:.65rem;}
.ll-ta{font-size:.74rem;font-weight:600;color:#a89cf8;}

/* PRICING — white */
.ll-pricing{background:#fff;padding:5rem 2rem;text-align:center;}
.ll-pricing-inner{max-width:1100px;margin:0 auto;}
.ll-price-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));gap:1.1rem;margin-top:2.5rem;}
.ll-price-card{background:#F8FAFC;border:2px solid #e5e7eb;border-radius:16px;padding:2rem;text-align:left;transition:all .2s;}
.ll-price-card.popular{border-color:#D4AF37;background:#fffbf0;}
.ll-price-card:hover{transform:translateY(-3px);box-shadow:0 12px 32px rgba(0,0,0,.08);}
.ll-price-tier{font-size:.62rem;letter-spacing:.12em;text-transform:uppercase;font-weight:700;margin-bottom:.5rem;color:#6b7280;}
.ll-price-tier.popular{color:#D4AF37;}
.ll-price-amt{font-family:'Cormorant Garamond',serif;font-size:3rem;font-weight:700;color:#111827;line-height:1;}
.ll-price-per{font-size:.78rem;color:#6b7280;margin-bottom:1.25rem;}
.ll-price-feat{list-style:none;font-size:.82rem;color:#374151;line-height:2;}
.ll-price-feat li::before{content:'✓ ';color:#16A34A;font-weight:700;}
.ll-price-btn{display:block;text-align:center;margin-top:1.25rem;padding:.78rem;border-radius:10px;
  font-weight:700;font-size:.88rem;text-decoration:none;transition:all .18s;}
.ll-price-btn-gold{background:#D4AF37;color:#0B1724;}
.ll-price-btn-gold:hover{background:#e8c84a;}
.ll-price-btn-outline{background:transparent;color:#1E3A8A;border:2px solid #1E3A8A;}
.ll-price-btn-outline:hover{background:#1E3A8A;color:#fff;}

/* DISCLAIMER — light */
.ll-disclaimer{background:#F4F6F8;border-top:1px solid #e5e7eb;padding:2rem;text-align:center;}
.ll-disclaimer p{font-size:.76rem;color:#6b7280;max-width:760px;margin:0 auto;line-height:1.7;}
.ll-disclaimer strong{color:#374151;}

/* CTA — dark */
.ll-cta{background:#0B1724;padding:5rem 2rem;text-align:center;
  border-top:1px solid rgba(212,175,55,.15);}
.ll-cta-t{font-family:'Cormorant Garamond',serif;font-size:2.6rem;font-weight:600;
  color:#f8fafc;margin-bottom:.65rem;}
.ll-cta-s{font-size:.9rem;color:#64748b;margin-bottom:1.75rem;}
</style>

<div class="ll-page">

<!-- NAV -->
<nav class="ll-nav">
  <a class="ll-brand-wrap" href="/">
    <div class="ll-brand-ic">LL</div>
    <span class="ll-brand-txt">LegacyLift</span>
  </a>
  <div class="ll-nav-r">
    <a href="/pricing" class="ll-nbtn">Pricing</a>
    <a href="/about" class="ll-nbtn">Our Story</a>
    <a href="/login" class="ll-nbtn">Login</a>
    <a href="/signup" class="ll-nbtn-cta">Start Free &rarr;</a>
  </div>
</nav>

<!-- HERO — dark navy -->
<div class="ll-hero">
  <div class="ll-pill">Generational Wealth Platform</div>
  <h1 class="ll-h1">
    <span class="ll-gold">Lift your family.</span><br/>
    <span class="ll-lift">Build your legacy.</span>
  </h1>
  <p class="ll-sub">LegacyLift helps everyday families find money opportunities they may be missing.</p>
  <p class="ll-sub2">Access simple tools, financial education, and tax-break guidance designed to help you protect more of what you earn and build long-term stability.</p>
  <div class="ll-btns">
    <a href="/signup" class="ll-btn-gold">Start Using LegacyLift &rarr;</a>
    <a href="/pricing" class="ll-btn-ghost">Explore the Tools</a>
  </div>
  <div class="ll-trust">
    <span>Free to start</span><span>21 curated lessons</span><span>Adults + Children</span><span>No bank login required</span>
  </div>
</div>

<!-- STRIP — dark -->
<div class="ll-strip">
  <div class="ll-si"><strong>Freedom Score</strong> 0&ndash;100 composite</div>
  <div class="ll-si"><strong>Debt Engine</strong> DOLP &middot; Avalanche &middot; Snowball</div>
  <div class="ll-si"><strong>Cashflow</strong> CSV auto-analysis</div>
  <div class="ll-si"><strong>Retirement</strong> Milestone projections</div>
  <div class="ll-si"><strong>Education</strong> 21 lessons + quizzes</div>
</div>

<!-- HEADLINE — light -->
<div class="ll-headline">
  <h2 class="ll-hl-title">There is a gap between what some people know about money &mdash; and what most of us were ever taught. LegacyLift closes that gap.</h2>
  <p class="ll-hl-sub">There are tax strategies, investment accounts, and wealth-building tools that have always existed. Most families just never had access to them. Until now.</p>
  <a href="/signup" class="ll-hl-cta">Get Started Free &mdash; No Credit Card</a>
</div>

<!-- WHAT YOU GET — light gray -->
<div class="ll-wyg">
  <div class="ll-wyg-inner">
    <div style="text-align:center;margin-bottom:1rem;">
      <div class="ll-wyg-tag">What You Get</div>
      <div class="ll-wyg-title">Four things that change how your family handles money.</div>
      <div class="ll-wyg-sub">Simple. Practical. Built for real families.</div>
    </div>
    <div class="ll-wyg-grid">
      <div class="ll-wyg-card">
        <div class="ll-wyg-ic">&#x1F4B0;</div>
        <div class="ll-wyg-ct">Tax Break Guidance</div>
        <div class="ll-wyg-cd">Learn about deductions, credits, and opportunities people often overlook &mdash; explained simply, without confusing financial language.</div>
      </div>
      <div class="ll-wyg-card">
        <div class="ll-wyg-ic">&#x1F4CA;</div>
        <div class="ll-wyg-ct">Financial Tools</div>
        <div class="ll-wyg-cd">Use simple tools to organize your money, track your debt, plan your retirement, and see your financial health in one number.</div>
      </div>
      <div class="ll-wyg-card">
        <div class="ll-wyg-ic">&#x1F4DA;</div>
        <div class="ll-wyg-ct">AI-Powered Education</div>
        <div class="ll-wyg-cd">Get beginner-friendly explanations of financial concepts &mdash; adults AND children. 21 curated lessons with quizzes and progress tracking.</div>
      </div>
      <div class="ll-wyg-card">
        <div class="ll-wyg-ic">&#x1F3DB;</div>
        <div class="ll-wyg-ct">Legacy Planning Mindset</div>
        <div class="ll-wyg-cd">Build smarter habits for your family, your future, and long-term growth. Generational wealth starts with one household deciding to think differently.</div>
      </div>
    </div>
  </div>
</div>

<!-- TOOLS — white -->
<div class="ll-tools">
  <div class="ll-tools-inner">
    <div style="text-align:center;margin-bottom:2.5rem;">
      <div class="ll-wyg-tag">Platform Tools</div>
      <div class="ll-wyg-title">Real tools. Not just another course.</div>
      <div class="ll-wyg-sub">Every tool is built in. No extra subscriptions. No upsells.</div>
    </div>
    <div class="ll-tools-grid">
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F4CA;</div>
        <div class="ll-tool-t">Freedom Score (0&ndash;100)</div>
        <div class="ll-tool-d">Your real-time composite financial health score across cashflow, emergency fund, debt, investing, and spending leaks.</div>
      </a>
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F4C1;</div>
        <div class="ll-tool-t">Cashflow Analyzer</div>
        <div class="ll-tool-d">Upload your bank CSV. Every transaction categorized. Spending leaks surfaced in 60 seconds. No bank login required.</div>
      </a>
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F4A3;</div>
        <div class="ll-tool-t">Debt Destruction Engine</div>
        <div class="ll-tool-d">DOLP, Avalanche, or Snowball. Exact month-by-month payoff schedule showing your precise debt-free date.</div>
      </a>
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F4C8;</div>
        <div class="ll-tool-t">Retirement Planner</div>
        <div class="ll-tool-d">Future value projections with decade milestones and 4%-rule income estimates. See what your investing becomes in 20 years.</div>
      </a>
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F9FE;</div>
        <div class="ll-tool-t">Budget Planner</div>
        <div class="ll-tool-d">Give every dollar a job. Track category performance. See exactly where your money goes every month.</div>
      </a>
      <a href="/signup" class="ll-tool-card">
        <div class="ll-tool-ic">&#x1F4DA;</div>
        <div class="ll-tool-t">21-Lesson Curriculum</div>
        <div class="ll-tool-d">Adult lessons, children's lessons (ages 5&ndash;14), and 9 tax strategy lessons. Quizzes and progress tracking included.</div>
      </a>
    </div>
  </div>
</div>

<!-- INSIDE LEGACYLIFT — light -->
<div class="ll-inside">
  <div class="ll-inside-inner">
    <div style="text-align:center;margin-bottom:.5rem;">
      <div class="ll-wyg-tag">Inside LegacyLift</div>
      <div class="ll-wyg-title">See what's waiting for you.</div>
      <div class="ll-wyg-sub">Everything is built in and ready to use the moment you sign up.</div>
    </div>
    <div class="ll-inside-grid">
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x1F3AF;</div><div class="ll-inside-t">Tax Break Finder</div><div class="ll-inside-d">Strategies most families never learn</div></div>
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x2705;</div><div class="ll-inside-t">Deduction Checklist</div><div class="ll-inside-d">Know what you can keep</div></div>
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x1F46A;</div><div class="ll-inside-t">Family Financial Planner</div><div class="ll-inside-d">Adults and children together</div></div>
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x1F4BC;</div><div class="ll-inside-t">Side Hustle Tax Guide</div><div class="ll-inside-d">Solo 401k, deductions, and more</div></div>
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x1F4C5;</div><div class="ll-inside-t">Monthly Money Tracker</div><div class="ll-inside-d">Budget, cashflow, and score</div></div>
      <div class="ll-inside-card"><div class="ll-inside-ic">&#x1F4B8;</div><div class="ll-inside-t">Freedom Score Dashboard</div><div class="ll-inside-d">Your financial health in one number</div></div>
    </div>
  </div>
</div>

<!-- WHY I BUILT IT — dark -->
<div class="ll-why">
  <div class="ll-why-inner">
    <span class="ll-why-tag">Why I Built LegacyLift</span>
    <div class="ll-why-title">Built to close the gap &mdash; not widen it.</div>
    <p class="ll-why-body">I built LegacyLift because too many everyday people are working hard but still missing access to information that could help them save, plan, and build. Some people have advisors, accountants, and tools around them. Most families do not. LegacyLift was created to close that gap by making financial and tax-break education easier to understand and easier to use.</p>
    <a href="/signup" class="ll-btn-gold" style="display:inline-block;padding:.82rem 2.2rem;font-size:.9rem;border-radius:10px;font-weight:700;text-decoration:none;background:#D4AF37;color:#0B1724;">Start for Free &rarr;</a>
  </div>
</div>

<!-- TESTIMONIALS — dark -->
<div class="ll-tests">
  <div style="text-align:center;margin-bottom:2.5rem;">
    <div class="ll-wyg-tag" style="color:#D4AF37;background:rgba(212,175,55,.1);border-color:rgba(212,175,55,.2);">Real Families</div>
    <div style="font-family:'Cormorant Garamond',serif;font-size:2rem;font-weight:600;color:#f8fafc;margin-top:.5rem;">Built for people like you.</div>
  </div>
  <div class="ll-tgrid">
    <div class="ll-tc"><div class="ll-ts">&#x2736; &#x2736; &#x2736; &#x2736; &#x2736;</div><div class="ll-tb">"The DOLP system worked exactly as described. Freedom Score went from 38 to 74 in under two years. This platform changes how you think about money."</div><div class="ll-ta">@marcus_t &mdash; Paid off $27K in 22 months</div></div>
    <div class="ll-tc"><div class="ll-ts">&#x2736; &#x2736; &#x2736; &#x2736; &#x2736;</div><div class="ll-tb">"We did the children's curriculum together. My daughter now asks 'need or want?' before every purchase &mdash; completely unprompted. That's a generational win."</div><div class="ll-ta">@priya_k &mdash; My daughter manages money at 10</div></div>
    <div class="ll-tc"><div class="ll-ts">&#x2736; &#x2736; &#x2736; &#x2736; &#x2736;</div><div class="ll-tb">"I'm 44. Raising my monthly investing from $200 to $550 produces $280K more at 65. The retirement planner showed me what I was leaving behind."</div><div class="ll-ta">@james_r &mdash; Retirement strategy transformed</div></div>
  </div>
</div>

<!-- PRICING — white -->
<div class="ll-pricing">
  <div class="ll-pricing-inner">
    <div class="ll-wyg-tag">Pricing</div>
    <div class="ll-wyg-title">Simple, honest pricing.</div>
    <div class="ll-wyg-sub">Start free. Upgrade when you're ready. No tricks.</div>
    <div class="ll-price-grid">
      <div class="ll-price-card">
        <div class="ll-price-tier">Free</div>
        <div class="ll-price-amt">$0</div>
        <div class="ll-price-per">Forever free &mdash; no card required</div>
        <ul class="ll-price-feat">
          <li>Freedom Score (0&ndash;100)</li>
          <li>Budget Planner</li>
          <li>Community Feed</li>
          <li>5 Foundation Lessons</li>
        </ul>
        <a href="/signup" class="ll-price-btn ll-price-btn-outline">Get Started Free</a>
      </div>
      <div class="ll-price-card popular">
        <div class="ll-price-tier popular">&#x2736; Most Popular</div>
        <div class="ll-price-amt" style="color:#D4AF37;">$9.99</div>
        <div class="ll-price-per">per month &middot; cancel anytime</div>
        <ul class="ll-price-feat">
          <li>Everything in Free</li>
          <li>CSV Cashflow Analyzer</li>
          <li>Debt Destruction Engine</li>
          <li>Retirement Projection</li>
          <li>All 21 Education Lessons</li>
          <li>Children's Curriculum</li>
          <li>Weekly Score Report Email</li>
        </ul>
        <a href="/signup" class="ll-price-btn ll-price-btn-gold">Start Monthly Plan</a>
      </div>
      <div class="ll-price-card">
        <div class="ll-price-tier">Lifetime</div>
        <div class="ll-price-amt">$49</div>
        <div class="ll-price-per">one-time &middot; yours forever</div>
        <ul class="ll-price-feat">
          <li>Everything in Monthly</li>
          <li>Pay once, never again</li>
          <li>All future lessons included</li>
          <li>B2B org features</li>
          <li>Best value for families</li>
        </ul>
        <a href="/signup" class="ll-price-btn ll-price-btn-outline">Get Lifetime Access</a>
      </div>
    </div>
  </div>
</div>

<!-- DISCLAIMER — light -->
<div class="ll-disclaimer">
  <p><strong>Disclaimer:</strong> LegacyLift provides educational tools and general financial guidance. It does not replace professional tax, legal, or financial advice. Always consult a qualified tax professional before making financial decisions. Results vary based on individual financial situations.</p>
</div>

<!-- CTA — dark -->
<div class="ll-cta">
  <div class="ll-cta-t">Your legacy begins with one decision.</div>
  <div class="ll-cta-s">Free to start. No credit card. No bank connection required.</div>
  <a href="/signup" class="ll-btn-gold" style="display:inline-block;padding:.9rem 2.5rem;font-size:1rem;border-radius:10px;font-weight:700;text-decoration:none;background:#D4AF37;color:#0B1724;box-shadow:0 4px 16px rgba(212,175,55,.35);">Create Your Free Account &rarr;</a>
</div>

</div>"""
    return page("LegacyLift &mdash; Lift Your Family. Build Your Legacy.", body)

@app.get("/signup")

@app.get("/signup")
def signup():
    body = f"""<div style="max-width:420px;margin:3rem auto;">
<div class="section-label">Create Account</div><h1 style="margin-bottom:1.5rem;">Get started free</h1>
<div class="card"><form method="post" action="{url_for('signup_post')}">
<div class="form-group"><label>Email</label><input type="email" name="email" required autofocus></div>
<div class="form-group"><label>Password (min 6 chars)</label><input type="password" name="password" minlength="6" required></div>
<button class="btn btn-primary btn-block" type="submit">Create Account →</button>
</form><hr><p class="muted" style="font-size:.8rem;text-align:center;">Already have an account? <a href="{url_for('login')}">Login</a></p>
</div></div>"""
    return page("Sign Up", body)

@app.post("/signup")
def signup_post():
    email = (request.form.get("email") or "").strip().lower()
    pw    = request.form.get("password") or ""
    if not email or not pw: flash("Email and password required.", "warning"); return redirect(url_for("signup"))
    if User.query.filter_by(email=email).first(): flash("Account exists — please login.", "warning"); return redirect(url_for("login"))
    u = User(email=email); u.set_password(pw)
    db.session.add(u); db.session.commit(); ensure_profile(u)
    session["uid"] = u.id; flash("Account created. Welcome!", "success")
    return redirect(url_for("onboarding"))

@app.get("/login")
def login():
    body = f"""<div style="max-width:420px;margin:3rem auto;">
<div class="section-label">Authentication</div><h1 style="margin-bottom:1.5rem;">Login</h1>
<div class="card"><form method="post" action="{url_for('login_post')}">
<div class="form-group"><label>Email</label><input type="email" name="email" required autofocus></div>
<div class="form-group"><label>Password</label><input type="password" name="password" required></div>
<button class="btn btn-primary btn-block" type="submit">Login →</button>
</form><hr><div style="display:flex;justify-content:space-between;font-size:.8rem;"><a href="{url_for('forgot_password')}" style="color:var(--muted);">Forgot password?</a><a href="{url_for('signup')}">Create account →</a></div>
</div></div>"""
    return page("Login", body)

@app.post("/login")
def login_post():
    email = (request.form.get("email") or "").strip().lower()
    pw    = request.form.get("password") or ""
    u = User.query.filter_by(email=email).first()
    if not u or not u.check_password(pw): flash("Invalid credentials.", "warning"); return redirect(url_for("login"))
    session["uid"] = u.id; flash("Welcome back.", "success")
    return redirect(url_for("dashboard"))

@app.get("/logout")
def logout():
    session.clear(); flash("Logged out.", "success"); return redirect(url_for("home"))

# ── Stripe Payment Routes ─────────────────────────────────────────────────────

@app.get("/pricing")
def pricing():
    u = me()
    active = premium_gate(u) if u else False
    monthly_btn = f'<a class="btn btn-primary btn-block" href="{url_for("checkout", plan="monthly")}">Subscribe — $9.99/mo →</a>'
    lifetime_btn= f'<a class="btn btn-block" style="border-color:var(--amber);color:var(--amber);" href="{url_for("checkout", plan="lifetime")}">Buy Once — $49 Lifetime →</a>'
    if not STRIPE_ENABLED:
        monthly_btn = f'<a class="btn btn-primary btn-block" href="{url_for("signup")}">Get Started →</a>'
        lifetime_btn= f'<a class="btn btn-block btn-amber" href="{url_for("signup")}">Get Started →</a>'

    active_html = '<div class="alert-item ok" style="margin-bottom:1rem;text-align:center;">✓ You already have Premium access.</div>' if active else ""
    body = f"""
<div style="max-width:800px;margin:0 auto;">
  <div style="text-align:center;margin-bottom:2.5rem;">
    <div class="section-label" style="justify-content:center;">Pricing</div>
    <h1 style="font-size:2rem;margin-bottom:.5rem;">Simple, honest pricing</h1>
    <p class="muted">Start free. Upgrade when you're ready. No tricks.</p>
  </div>
  {active_html}
  <div class="grid grid-3" style="gap:1.25rem;align-items:start;">

    <div class="card">
      <div class="section-label">Free</div>
      <div style="font-family:var(--mono);font-size:2.2rem;font-weight:700;color:#d8edd8;margin-bottom:.25rem;">$0</div>
      <div class="muted" style="font-size:.82rem;margin-bottom:1.25rem;">Forever free</div>
      <ul style="list-style:none;font-size:.84rem;line-height:2.2;color:var(--text);margin-bottom:1.5rem;">
        <li>✓ Freedom Score (0–100)</li>
        <li>✓ Budget Planner</li>
        <li>✓ Community Feed</li>
        <li>✓ 5 Free Education Lessons</li>
        <li style="color:var(--muted);">✗ CSV Cashflow Analyzer</li>
        <li style="color:var(--muted);">✗ Debt Engine</li>
        <li style="color:var(--muted);">✗ Retirement Projection</li>
        <li style="color:var(--muted);">✗ Premium Lessons (7)</li>
      </ul>
      <a class="btn btn-block" href="{url_for('signup')}">Get Started Free →</a>
    </div>

    <div class="card" style="border-color:var(--green);position:relative;">
      <div style="position:absolute;top:-12px;left:50%;transform:translateX(-50%);
                  background:var(--green);color:var(--bg);font-family:var(--mono);font-size:.68rem;
                  font-weight:700;padding:.2rem .75rem;border-radius:20px;white-space:nowrap;">MOST POPULAR</div>
      <div class="section-label" style="color:var(--green);">Monthly</div>
      <div style="font-family:var(--mono);font-size:2.2rem;font-weight:700;color:var(--green);margin-bottom:.25rem;">$9.99</div>
      <div class="muted" style="font-size:.82rem;margin-bottom:1.25rem;">per month · cancel anytime</div>
      <ul style="list-style:none;font-size:.84rem;line-height:2.2;color:var(--text);margin-bottom:1.5rem;">
        <li>✓ Everything in Free</li>
        <li>✓ CSV Cashflow Analyzer</li>
        <li>✓ Debt Destruction Engine</li>
        <li>✓ Retirement Projection</li>
        <li>✓ All 12 Education Lessons</li>
        <li>✓ Kids Curriculum (Premium)</li>
        <li>✓ Priority Support</li>
      </ul>
      {monthly_btn}
    </div>

    <div class="card" style="border-color:var(--amber);">
      <div class="section-label" style="color:var(--amber);">Lifetime</div>
      <div style="font-family:var(--mono);font-size:2.2rem;font-weight:700;color:var(--amber);margin-bottom:.25rem;">$49</div>
      <div class="muted" style="font-size:.82rem;margin-bottom:1.25rem;">one-time · yours forever</div>
      <ul style="list-style:none;font-size:.84rem;line-height:2.2;color:var(--text);margin-bottom:1.5rem;">
        <li>✓ Everything in Monthly</li>
        <li>✓ Pay once, never again</li>
        <li>✓ All future lessons included</li>
        <li>✓ B2B org features</li>
        <li>✓ Best value for families</li>
        <li>&nbsp;</li>
        <li>&nbsp;</li>
      </ul>
      {lifetime_btn}
    </div>
  </div>

  <div class="card" style="margin-top:1.5rem;text-align:center;">
    <div class="section-label" style="justify-content:center;">B2B / Organizations</div>
    <p class="muted" style="font-size:.85rem;margin-bottom:1rem;">
      Employers, credit unions, and schools — license seats for your team or members.
    </p>
    <a class="btn" href="{url_for('org')}">Organization Portal →</a>
  </div>
</div>"""
    return page("Pricing", body)

@app.get("/checkout/<plan>")
def checkout(plan):
    gate = require_login()
    if gate: return gate
    u = me()
    if premium_gate(u): flash("You already have Premium access.", "success"); return redirect(url_for("dashboard"))
    if not STRIPE_ENABLED:
        flash("Stripe is not yet configured. Use a license key for now.", "warning")
        return redirect(url_for("premium"))
    if plan not in ("monthly", "lifetime"):
        abort(400)
    price_id = STRIPE_MONTHLY_PRICE if plan == "monthly" else STRIPE_LIFETIME_PRICE
    if not price_id:
        flash("Payment plan not configured yet.", "warning"); return redirect(url_for("pricing"))
    try:
        # Get or create Stripe customer
        if not u.stripe_customer_id:
            customer = stripe.Customer.create(email=u.email, metadata={"user_id": u.id})
            u.stripe_customer_id = customer.id; db.session.commit()
        mode = "subscription" if plan == "monthly" else "payment"
        checkout_session = stripe.checkout.Session.create(
            customer=u.stripe_customer_id,
            payment_method_types=["card"],
            line_items=[{"price": price_id, "quantity": 1}],
            mode=mode,
            success_url=APP_URL + url_for("checkout_success") + "?session_id={CHECKOUT_SESSION_ID}",
            cancel_url=APP_URL + url_for("pricing"),
            metadata={"user_id": u.id, "plan": plan},
        )
        return redirect(checkout_session.url, code=303)
    except Exception as e:
        flash(f"Payment error: {str(e)}", "warning"); return redirect(url_for("pricing"))

@app.get("/checkout/success")
def checkout_success():
    gate = require_login()
    if gate: return gate
    flash("Payment successful! Premium is now active. Welcome to the full platform.", "success")
    return redirect(url_for("dashboard"))

@app.post("/stripe/webhook")
def stripe_webhook():
    """Stripe sends events here. Handles subscription + payment completion."""
    if not STRIPE_ENABLED: return jsonify({"status": "disabled"}), 200
    payload = request.get_data()
    sig_header = request.headers.get("Stripe-Signature", "")
    try:
        event = stripe.Webhook.construct_event(payload, sig_header, STRIPE_WEBHOOK_SECRET)
    except Exception:
        return jsonify({"error": "Invalid signature"}), 400

    etype = event["type"]

    if etype == "checkout.session.completed":
        sess = event["data"]["object"]
        user_id = int(sess.get("metadata", {}).get("user_id", 0))
        plan    = sess.get("metadata", {}).get("plan", "monthly")
        u = db.session.get(User, user_id)
        if u:
            u.is_premium = True
            if plan == "monthly":
                u.stripe_subscription_id = sess.get("subscription")
                u.subscription_status    = "active"
            else:
                u.subscription_status = "lifetime"
            db.session.commit()

    elif etype in ("customer.subscription.deleted", "customer.subscription.paused"):
        sub = event["data"]["object"]
        u = User.query.filter_by(stripe_subscription_id=sub["id"]).first()
        if u and u.subscription_status != "lifetime":
            u.is_premium = False; u.subscription_status = "canceled"; db.session.commit()

    elif etype == "customer.subscription.updated":
        sub = event["data"]["object"]
        u = User.query.filter_by(stripe_subscription_id=sub["id"]).first()
        if u:
            u.subscription_status = sub.get("status", "active")
            u.is_premium = sub.get("status") == "active"; db.session.commit()

    elif etype == "invoice.payment_failed":
        inv = event["data"]["object"]
        u = User.query.filter_by(stripe_customer_id=inv.get("customer")).first()
        if u and u.subscription_status != "lifetime":
            u.is_premium = False; u.subscription_status = "past_due"; db.session.commit()

    return jsonify({"status": "ok"}), 200

# ── Premium (manual key fallback) ─────────────────────────────────────────────

@app.get("/premium")
def premium():
    gate = require_login()
    if gate: return gate
    u = me(); active = premium_gate(u)
    sub_info = ""
    if u.subscription_status == "lifetime": sub_info = '<span class="badge badge-amber">Lifetime</span>'
    elif u.subscription_status == "active": sub_info = '<span class="badge badge-green">Monthly Active</span>'
    elif u.subscription_status == "past_due": sub_info = '<span class="badge" style="border-color:var(--red);color:var(--red);">Past Due</span>'
    body = f"""
<div class="section-label">Premium Access</div><h1 style="margin-bottom:1.5rem;">Your Plan {sub_info}</h1>
<div class="grid grid-2" style="gap:1rem;max-width:760px;">
  <div class="card" style="{'border-color:var(--green);' if active else ''}">
    {'<div class="alert-item ok" style="margin-bottom:1rem;">Premium is active.</div>' if active else '<div class="alert-item warn" style="margin-bottom:1rem;">Free tier — upgrade to unlock everything.</div>'}
    <div style="display:flex;gap:.75rem;flex-wrap:wrap;margin-bottom:1rem;">
      <a class="btn btn-primary" href="{url_for('pricing')}">View Plans & Pricing →</a>
    </div>
    <hr>
    <div class="section-label">Manual License Key</div>
    <form method="post" action="{url_for('premium_unlock')}">
      <div class="form-group"><label>Premium Key</label>
        <input type="text" name="key" class="mono" placeholder="FFOS-XXXX-XXXX" {'disabled' if active else ''} required>
      </div>
      <button class="btn btn-block" {'disabled' if active else ''} type="submit">Unlock with Key →</button>
    </form>
  </div>
  <div class="card">
    <div class="section-label">What's Included</div>
    <ul style="list-style:none;font-size:.84rem;line-height:2.2;color:var(--text);">
      <li>✓ All 12 Education Lessons (Adults + Kids)</li>
      <li>✓ CSV Cashflow Analyzer</li>
      <li>✓ Debt Destruction Engine</li>
      <li>✓ Retirement Projection</li>
      <li>✓ Full transaction history</li>
      <li>✓ B2B Organization access</li>
    </ul>
    <hr><a class="btn btn-block" href="{url_for('org')}">Organization / B2B Portal →</a>
  </div>
</div>"""
    return page("Premium", body)

@app.post("/premium/unlock")
def premium_unlock():
    gate = require_login()
    if gate: return gate
    u = me(); key = (request.form.get("key") or "").strip()
    pk = PremiumKey.query.filter_by(key=key, is_used=False).first()
    if not pk: flash("Invalid or already-used key.", "warning"); return redirect(url_for("premium"))
    pk.is_used = True; pk.used_by_user_id = u.id
    u.is_premium = True; u.premium_key = key; u.subscription_status = "lifetime"
    db.session.commit(); flash("Premium unlocked!", "success")
    return redirect(url_for("dashboard"))

# ── Dashboard ─────────────────────────────────────────────────────────────────

@app.get("/dashboard")
def dashboard():
    gate = require_login()
    if gate: return gate
    u = me(); p = ensure_profile(u)
    # Redirect new users to onboarding wizard
    if needs_onboarding(u):
        return redirect(url_for("onboarding"))
    score, parts = compute_freedom_score(u); alerts = generate_alerts(u)
    # Save daily score snapshot (once per day)
    today_date = dt.date.today()
    last_snap = ScoreHistory.query.filter_by(user_id=u.id)\
        .filter(db.func.date(ScoreHistory.recorded_at) == today_date).first()
    if not last_snap:
        db.session.add(ScoreHistory(user_id=u.id, score=score))
        db.session.commit()
    # Daily tip
    tip_idx  = today_date.timetuple().tm_yday % len(DAILY_TIPS)
    daily_tip = DAILY_TIPS[tip_idx]
    since = dt.date.today() - dt.timedelta(days=30)
    txs   = Transaction.query.filter(Transaction.user_id==u.id, Transaction.date>=since).all()
    spend_30  = sum(-t.amount for t in txs if t.amount<0)
    income_30 = sum(t.amount  for t in txs if t.amount>0)
    net_30    = income_30 - spend_30
    score_color = "var(--green)" if score>=60 else "var(--amber)" if score>=35 else "var(--red)"
    tier = '<span class="badge badge-green">PRO</span>' if premium_gate(u) else '<span class="badge badge-muted">FREE</span>'
    all_edu  = EduLesson.query.all()
    done_edu = EduProgress.query.filter_by(user_id=u.id, completed=True).count()
    alerts_html = "".join(f'<div class="alert-item {"warn" if a.startswith("⚠") else "ok" if a.startswith("✓") else ""}">{safe(a)}</div>' for a in alerts)
    score_parts_html = ""
    for label, key, total in [("Net Cashflow","net_points",25),("Emergency Fund","ef_points",20),
                               ("Debt Burden","debt_points",20),("Investing","inv_points",15),("Spend Leaks","leak_points",20)]:
        pts = parts.get(key,0); pct = int(pts/total*100)
        bar_cls = "bar-green" if pct>=70 else "bar-amber" if pct>=35 else "bar-red"
        score_parts_html += f'<div style="margin-bottom:.75rem;"><div style="display:flex;justify-content:space-between;margin-bottom:.3rem;"><span class="mono" style="font-size:.75rem;color:var(--muted);">{label}</span><span class="mono" style="font-size:.75rem;">{pts:.1f}/{total}</span></div><div class="bar-wrap"><div class="bar {bar_cls}" style="width:{pct}%;"></div></div></div>'
    body = f"""
<div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:1.5rem;flex-wrap:wrap;gap:.75rem;">
  <div><div class="section-label">Dashboard</div><h1>{safe(p.display_name)} {tier}</h1></div>
  <div style="display:flex;gap:.5rem;flex-wrap:wrap;">
    <a class="btn btn-sm" href="{url_for('scoreboard')}">Update Inputs</a>
    <a class="btn btn-sm" href="{url_for('upload_csv')}">Upload CSV</a>
    <a class="btn btn-sm btn-primary" href="{url_for('edu_learn')}">📚 Learn</a>
  </div>
</div>
{_daily_tip_card(daily_tip)}
<div class="grid grid-4" style="gap:1rem;margin-bottom:1.5rem;">
  <div class="stat-box"><div class="val" style="color:{score_color};font-size:2rem;">{score}<span style="font-size:1rem;color:var(--muted);">/100</span></div><div class="lbl">Freedom Score</div></div>
  <div class="stat-box"><div class="val" style="color:{'var(--green)' if net_30>=0 else 'var(--red)'};">{money(net_30)}</div><div class="lbl">30-Day Net</div></div>
  <div class="stat-box"><div class="val">{money(income_30)}</div><div class="lbl">30-Day Income</div></div>
  <div class="stat-box"><div class="val green">{done_edu}/{len(all_edu)}<span style="font-size:.85rem;color:var(--muted);"> lessons</span></div><div class="lbl">Education Progress</div></div>
</div>
<div class="grid grid-2" style="gap:1rem;">
  <div>
    <div class="card" style="margin-bottom:1rem;"><div class="section-label">Score Breakdown</div>{score_parts_html}</div>
    <div class="card"><div class="section-label">Quick Access</div>
      <div class="grid grid-2" style="gap:.5rem;margin-top:.25rem;">
        <a class="btn btn-block" href="{url_for('transactions')}">Cashflow</a>
        <a class="btn btn-block" href="{url_for('debt')}">Debt Engine</a>
        <a class="btn btn-block" href="{url_for('retirement')}">Retirement</a>
        <a class="btn btn-block btn-primary" href="{url_for('edu_learn')}">📚 Education</a>
      </div>
    </div>
  </div>
  <div class="card"><div class="section-label">Behavior Alerts</div>{alerts_html}</div>
</div>"""
    return page("Dashboard", body)

# ── Education Routes ──────────────────────────────────────────────────────────

@app.get("/learn/education")
def edu_learn():
    gate = require_login()
    if gate: return gate
    u = me()
    audience = request.args.get("audience", "adult")
    if audience not in ("adult","kids"): audience = "adult"
    lessons  = EduLesson.query.filter_by(audience=audience).order_by(EduLesson.order_index).all()
    prog_map = {ep.lesson_id: ep for ep in EduProgress.query.filter_by(user_id=u.id).all()}
    all_lessons = EduLesson.query.all()
    done_total  = EduProgress.query.filter_by(user_id=u.id, completed=True).count()
    done_aud    = sum(1 for l in lessons if prog_map.get(l.id) and prog_map[l.id].completed)
    pct_aud     = int(done_aud/len(lessons)*100) if lessons else 0
    from collections import OrderedDict
    modules = OrderedDict()
    for l in lessons: modules.setdefault(l.module, []).append(l)
    toggle = f"""<div style="display:flex;gap:0;border:1px solid var(--border);border-radius:3px;overflow:hidden;margin-left:auto;">
  <a href="{url_for('edu_learn')}?audience=adult" style="font-family:var(--mono);font-size:.78rem;padding:.45rem 1.25rem;text-decoration:none;background:{'var(--green)' if audience=='adult' else 'transparent'};color:{'var(--bg)' if audience=='adult' else 'var(--muted)'};font-weight:{'700' if audience=='adult' else '400'};">Adult</a>
  <a href="{url_for('edu_learn')}?audience=kids"  style="font-family:var(--mono);font-size:.78rem;padding:.45rem 1.25rem;text-decoration:none;background:{'var(--green)' if audience=='kids'  else 'transparent'};color:{'var(--bg)' if audience=='kids'  else 'var(--muted)'};font-weight:{'700' if audience=='kids' else '400'};">Kids</a>
</div>"""
    modules_html = ""
    for mod_name, mod_lessons in modules.items():
        mod_done = sum(1 for l in mod_lessons if prog_map.get(l.id) and prog_map[l.id].completed)
        mod_pct  = int(mod_done/len(mod_lessons)*100) if mod_lessons else 0
        bar_cls  = "bar-green" if mod_pct==100 else "bar-amber" if mod_pct>0 else "bar-red"
        cards = ""
        for l in mod_lessons:
            ep = prog_map.get(l.id); is_done = bool(ep and ep.completed); lscore = ep.score if ep else 0
            locked = l.is_premium and not premium_gate(u)
            done_b = f'<span class="badge badge-green">✓ {lscore}%</span>' if is_done else ""
            age_b  = f'<span class="badge badge-amber">{safe(l.age_group)}</span>' if l.age_group else ""
            lock_h = """<div style="position:absolute;inset:0;background:rgba(8,14,10,.75);display:flex;align-items:center;justify-content:center;border-radius:4px;"><div style="text-align:center;"><div style="font-size:1.5rem;margin-bottom:.25rem;">🔒</div><div style="font-family:var(--mono);font-size:.72rem;color:var(--amber);">PREMIUM</div></div></div>""" if locked else ""
            action_url = url_for("edu_lesson", lid=l.id)
            cards += f"""<div onclick="{'void(0)' if locked else f"window.location='{action_url}'"}" style="background:var(--surface);border:1px solid {'rgba(0,232,122,.3)' if is_done else 'var(--border)'};border-radius:4px;padding:1.25rem;cursor:{'default' if locked else 'pointer'};transition:border-color .15s,transform .15s;position:relative;overflow:hidden;" onmouseover="this.style.transform='translateY(-2px)'" onmouseout="this.style.transform='none'">
  <div style="position:absolute;top:0;left:0;right:0;height:2px;background:{l.color};opacity:{'1' if is_done else '.4'};"></div>
  {lock_h}
  <div style="display:flex;justify-content:space-between;align-items:flex-start;margin-bottom:.75rem;">
    <span style="font-size:28px;">{l.icon}</span>
    <div style="display:flex;gap:.3rem;flex-wrap:wrap;justify-content:flex-end;">{done_b}{age_b}</div>
  </div>
  <div style="font-family:var(--mono);font-weight:700;font-size:.9rem;color:#d8edd8;margin-bottom:.3rem;">{safe(l.title)}</div>
  <div style="font-size:.76rem;color:var(--muted);font-style:italic;margin-bottom:.75rem;">{safe(l.tagline)}</div>
  <div style="font-size:.76rem;color:{l.color};font-family:var(--mono);">{'Review →' if is_done else ('🔒 Upgrade to unlock' if locked else 'Start lesson →')}</div>
</div>"""
        modules_html += f"""<div style="margin-bottom:1.75rem;">
  <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:.5rem;">
    <div><div class="section-label">{safe(mod_name)}</div><div style="font-family:var(--mono);font-size:.8rem;color:var(--muted);">{mod_done}/{len(mod_lessons)} complete</div></div>
    <div style="font-family:var(--mono);color:{'var(--green)' if mod_pct==100 else 'var(--text)'};font-weight:600;">{mod_pct}%</div>
  </div>
  <div class="bar-wrap" style="margin-bottom:.85rem;"><div class="bar {bar_cls}" style="width:{mod_pct}%;"></div></div>
  <div style="display:grid;grid-template-columns:repeat(auto-fill,minmax(260px,1fr));gap:.75rem;">{cards}</div>
</div>"""
    scores = EduProgress.query.filter_by(user_id=u.id, completed=True).all()
    avg_score = f"{sum(s.score for s in scores)//len(scores)}%" if scores else "—"
    body = f"""
<div style="display:flex;align-items:center;justify-content:space-between;margin-bottom:1.5rem;flex-wrap:wrap;gap:.75rem;">
  <div><div class="section-label">Money Education</div><h1>Financial Literacy Curriculum</h1></div>
  {toggle}
</div>
<div class="grid grid-4" style="gap:1rem;margin-bottom:1.75rem;">
  <div class="stat-box"><div class="val green">{done_total}/{len(all_lessons)}</div><div class="lbl">Total Complete</div></div>
  <div class="stat-box"><div class="val">{int(done_total/len(all_lessons)*100) if all_lessons else 0}%</div><div class="lbl">Overall Progress</div></div>
  <div class="stat-box"><div class="val">{done_aud}/{len(lessons)}</div><div class="lbl">{'Adult' if audience=='adult' else 'Kids'} Complete</div></div>
  <div class="stat-box"><div class="val amber">{avg_score}</div><div class="lbl">Avg Quiz Score</div></div>
</div>
{modules_html}
<div style="border-top:1px solid var(--border);padding-top:1rem;text-align:center;">
  <span style="font-family:var(--mono);font-size:.72rem;color:var(--muted);">// EDUCATION ONLY — NOT FINANCIAL ADVICE</span>
</div>"""
    return page("Education", body, active_page="learn")

@app.get("/learn/education/<int:lid>")
def edu_lesson(lid):
    gate = require_login()
    if gate: return gate
    u = me(); lesson = db.session.get(EduLesson, lid)
    if not lesson: abort(404)
    if lesson.is_premium and not premium_gate(u):
        flash("This lesson requires Premium.", "warning"); return redirect(url_for("pricing"))
    ep = EduProgress.query.filter_by(user_id=u.id, lesson_id=lid).first()
    is_done = bool(ep and ep.completed); lscore = ep.score if ep else 0
    content_paragraphs = json.loads(lesson.content_json)
    content_html = "".join(f'<p style="font-size:.88rem;line-height:1.8;color:var(--text);margin-bottom:.85rem;">{safe(p)}</p>' for p in content_paragraphs)
    siblings = EduLesson.query.filter_by(audience=lesson.audience).order_by(EduLesson.order_index).all()
    sib_ids  = [s.id for s in siblings]; idx = sib_ids.index(lid) if lid in sib_ids else -1
    prev_url = url_for("edu_lesson", lid=sib_ids[idx-1]) if idx>0 else None
    next_url = url_for("edu_lesson", lid=sib_ids[idx+1]) if idx<len(sib_ids)-1 else None
    prev_btn = f'<a class="btn" href="{prev_url}">← Previous</a>' if prev_url else ""
    next_btn = f'<a class="btn btn-primary" href="{next_url}">Next →</a>' if next_url else ""
    done_badge = f'<span class="badge badge-green">Completed — {lscore}%</span>' if is_done else ""
    quiz_html = ""
    if lesson.questions:
        q_html = ""
        for i, q in enumerate(lesson.questions):
            opts = "".join(f'<label style="display:flex;align-items:flex-start;gap:.6rem;padding:.5rem .75rem;margin-bottom:.3rem;border:1px solid var(--border);border-radius:3px;cursor:pointer;font-size:.82rem;color:var(--text);transition:border-color .15s;" onmouseover="this.style.borderColor=\'var(--green)\'" onmouseout="this.style.borderColor=\'var(--border)\'"><input type="radio" name="q_{q.id}" value="{letter}" required>&nbsp;<span><strong style="color:var(--muted);">{letter}.</strong> {safe(getattr(q,f"opt_{letter.lower()}"))}</span></label>' for letter in ["A","B","C","D"])
            q_html += f'<div style="margin-bottom:1.25rem;"><p style="font-family:var(--mono);font-weight:600;font-size:.85rem;margin-bottom:.65rem;color:#d8edd8;">Q{i+1}. {safe(q.prompt)}</p>{opts}</div>'
        quiz_html = f'<div class="card" style="margin-top:1.5rem;border-color:rgba(0,232,122,.3);"><div class="section-label">Knowledge Check</div><form method="post" action="{url_for("edu_quiz",lid=lid)}">{q_html}<button class="btn btn-primary btn-block" type="submit">Submit Quiz →</button></form></div>'
    body = f"""<div style="max-width:720px;margin:0 auto;">
  <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:.75rem;flex-wrap:wrap;gap:.5rem;">
    <a href="{url_for('edu_learn')}?audience={lesson.audience}" class="btn btn-sm">← Curriculum</a>
    <div style="display:flex;gap:.4rem;align-items:center;">
      <span class="badge {'badge-pink' if lesson.audience=='kids' else 'badge-green'}">{safe(lesson.audience).title()}</span>
      <span class="badge badge-muted">{safe(lesson.module)}</span>{done_badge}
    </div>
  </div>
  <div style="background:{lesson.color}18;border:1px solid {lesson.color}44;border-radius:4px;padding:1.75rem 1.5rem;margin-bottom:1rem;position:relative;overflow:hidden;">
    <div style="position:absolute;top:-20px;right:-10px;font-size:100px;opacity:.07;">{lesson.icon}</div>
    <div style="font-size:36px;margin-bottom:10px;">{lesson.icon}</div>
    <div style="font-family:var(--mono);font-size:1.4rem;font-weight:700;color:#d8edd8;margin-bottom:.3rem;line-height:1.1;">{safe(lesson.title)}</div>
    <div style="font-size:.85rem;color:var(--muted);font-style:italic;">{safe(lesson.tagline)}</div>
    {f'<div style="margin-top:.5rem;"><span class="badge badge-amber">{safe(lesson.age_group)}</span></div>' if lesson.age_group else ""}
  </div>
  <div class="card" style="margin-bottom:1rem;"><div class="section-label">lesson_content</div>{content_html}</div>
  <div style="background:{lesson.color}14;border:1px solid {lesson.color}44;border-radius:4px;padding:1rem 1.25rem;margin-bottom:1rem;display:flex;gap:1rem;align-items:center;">
    <div style="font-family:var(--mono);font-size:.68rem;letter-spacing:.1em;text-transform:uppercase;color:{lesson.color};flex-shrink:0;">The Numbers</div>
    <div style="font-family:var(--mono);font-size:.88rem;font-weight:600;color:{lesson.color};">{safe(lesson.stat)}</div>
  </div>
  <div class="card" style="margin-bottom:1rem;">
    <div class="section-label" style="color:var(--amber);">action_step</div>
    <p style="font-size:.88rem;line-height:1.7;color:var(--text);margin:0;">{safe(lesson.action)}</p>
  </div>
  {quiz_html}
  <div style="display:flex;justify-content:space-between;margin-top:1.5rem;flex-wrap:wrap;gap:.5rem;">
    <div>{prev_btn}</div>
    <div style="display:flex;gap:.5rem;"><a class="btn" href="{url_for('edu_learn')}?audience={lesson.audience}">All Lessons</a>{next_btn}</div>
  </div>
</div>"""
    return page(lesson.title, body, active_page="learn")

@app.post("/learn/education/<int:lid>/quiz")
def edu_quiz(lid):
    gate = require_login()
    if gate: return gate
    u = me(); lesson = db.session.get(EduLesson, lid)
    if not lesson: abort(404)
    if lesson.is_premium and not premium_gate(u): return redirect(url_for("pricing"))
    questions = lesson.questions
    if not questions: return redirect(url_for("edu_lesson", lid=lid))
    correct_count = 0; results = []
    for q in questions:
        answer = (request.form.get(f"q_{q.id}") or "").upper()
        ok = answer == q.correct.upper()
        if ok: correct_count += 1
        results.append({"prompt":q.prompt,"your":answer,"correct":q.correct.upper(),"ok":ok,
                        "explanation":q.explanation or "","opts":{"A":q.opt_a,"B":q.opt_b,"C":q.opt_c,"D":q.opt_d}})
    score = int(correct_count/len(questions)*100)
    ep = EduProgress.query.filter_by(user_id=u.id, lesson_id=lid).first()
    if not ep: ep = EduProgress(user_id=u.id, lesson_id=lid); db.session.add(ep)
    ep.completed = True; ep.score = max(ep.score, score); ep.updated_at = dt.datetime.utcnow()
    db.session.commit()
    siblings = EduLesson.query.filter_by(audience=lesson.audience).order_by(EduLesson.order_index).all()
    sib_ids = [s.id for s in siblings]; idx = sib_ids.index(lid) if lid in sib_ids else -1
    next_url = url_for("edu_lesson", lid=sib_ids[idx+1]) if idx<len(sib_ids)-1 else None
    next_btn = f'<a class="btn btn-primary" href="{next_url}">Next Lesson →</a>' if next_url else ""
    score_color = "var(--green)" if score>=80 else "var(--amber)" if score>=50 else "var(--red)"
    result_html = ""
    for r in results:
        cls = "ok" if r["ok"] else "warn"; icon = "+" if r["ok"] else "−"
        wrong = f' — Correct: <strong>{r["correct"]}. {safe(r["opts"].get(r["correct"],""))}</strong>' if not r["ok"] else ""
        expl  = f'<div style="font-size:.74rem;margin-top:.3rem;opacity:.8;">{safe(r["explanation"])}</div>' if r["explanation"] else ""
        result_html += f'<div class="alert-item {cls}" style="margin-bottom:.75rem;padding:1rem;"><div style="font-weight:600;margin-bottom:.35rem;">{icon} {safe(r["prompt"])}</div><div style="font-size:.78rem;">Your answer: <strong>{r["your"]}. {safe(r["opts"].get(r["your"],"?"))}</strong>{wrong}</div>{expl}</div>'
    body = f"""<div style="max-width:680px;margin:0 auto;">
  <div class="section-label">Quiz Results</div><h1 style="margin-bottom:1.5rem;">{safe(lesson.title)}</h1>
  <div class="card" style="text-align:center;margin-bottom:1.5rem;padding:2rem;">
    <div style="font-family:var(--mono);font-size:3.5rem;font-weight:600;color:{score_color};">{score}%</div>
    <div class="muted" style="margin-top:.25rem;">{correct_count}/{len(questions)} correct</div>
    <div style="color:{'var(--green)' if score>=80 else 'var(--amber)'};margin-top:.5rem;font-family:var(--mono);font-size:.82rem;">{'// LESSON COMPLETE' if score>=80 else '// REVIEW AND RETRY'}</div>
  </div>
  <div class="section-label">Answer Review</div>{result_html}
  <div style="display:flex;gap:.75rem;margin-top:1.5rem;flex-wrap:wrap;">
    <a class="btn" href="{url_for('edu_lesson',lid=lid)}">← Retake</a>
    <a class="btn" href="{url_for('edu_learn')}?audience={lesson.audience}">All Lessons</a>
    {next_btn}
  </div>
</div>"""
    return page("Quiz Results", body, active_page="learn")

# ── Scoreboard ────────────────────────────────────────────────────────────────

@app.get("/scoreboard")
def scoreboard():
    gate = require_login()
    if gate: return gate
    u = me(); p = ensure_profile(u); score, parts = compute_freedom_score(u); net = parts["net"]
    body = f"""<div class="section-label">Scoreboard</div><h1 style="margin-bottom:.25rem;">Score Inputs</h1>
<p class="muted" style="margin-bottom:1.5rem;">Update monthly. These drive your Freedom Score.</p>
<div class="grid grid-2" style="gap:1rem;">
  <div class="card"><div class="section-label">Current Score</div>
    <div class="mono" style="font-size:3rem;font-weight:600;color:{'var(--green)' if score>=60 else 'var(--amber)' if score>=35 else 'var(--red)'};">{score}<span style="font-size:1.2rem;color:var(--muted);">/100</span></div>
    <p class="muted" style="margin-top:.5rem;font-size:.82rem;">Net monthly: <span class="mono green">{money(net)}</span></p>
  </div>
  <div class="card"><form method="post" action="{url_for('scoreboard_save')}">
    <div class="grid grid-2" style="gap:.75rem;">
      <div class="form-group"><label>Monthly Income</label><input type="number" name="monthly_income" step="1" min="0" value="{p.monthly_income:.0f}"></div>
      <div class="form-group"><label>Fixed Bills</label><input type="number" name="fixed_bills" step="1" min="0" value="{p.fixed_bills:.0f}"></div>
      <div class="form-group"><label>Variable Spend</label><input type="number" name="variable_spend" step="1" min="0" value="{p.variable_spend:.0f}"></div>
      <div class="form-group"><label>Debt Minimums</label><input type="number" name="debt_minimums" step="1" min="0" value="{p.debt_minimums:.0f}"></div>
      <div class="form-group"><label>Emergency Fund ($)</label><input type="number" name="emergency_fund_current" step="1" min="0" value="{p.emergency_fund_current:.0f}"></div>
      <div class="form-group"><label>EF Target ($)</label><input type="number" name="emergency_fund_target" step="1" min="0" value="{p.emergency_fund_target:.0f}"></div>
      <div class="form-group"><label>Investing / mo</label><input type="number" name="monthly_investing_target" step="1" min="0" value="{p.monthly_investing_target:.0f}"></div>
      <div class="form-group"><label>Extra Debt / mo</label><input type="number" name="extra_debt_target" step="1" min="0" value="{p.extra_debt_target:.0f}"></div>
    </div>
    <button class="btn btn-primary btn-block" type="submit">Save →</button>
  </form></div>
</div>"""
    return page("Scoreboard", body)

@app.post("/scoreboard")
def scoreboard_save():
    gate = require_login()
    if gate: return gate
    u = me(); p = ensure_profile(u)
    def f(n):
        try: return float(request.form.get(n) or 0.0)
        except: return 0.0
    p.monthly_income=max(0.0,f("monthly_income")); p.fixed_bills=max(0.0,f("fixed_bills"))
    p.variable_spend=max(0.0,f("variable_spend")); p.debt_minimums=max(0.0,f("debt_minimums"))
    p.emergency_fund_current=max(0.0,f("emergency_fund_current")); p.emergency_fund_target=max(0.0,f("emergency_fund_target"))
    p.monthly_investing_target=max(0.0,f("monthly_investing_target")); p.extra_debt_target=max(0.0,f("extra_debt_target"))
    db.session.commit(); flash("Saved.", "success"); return redirect(url_for("scoreboard"))

# ── CSV / Transactions ────────────────────────────────────────────────────────

@app.get("/upload")
def upload_csv():
    gate = require_login()
    if gate: return gate
    u = me(); locked = not premium_gate(u)
    lock_html = f'<div class="alert-item warn" style="margin-bottom:1rem;">CSV Analyzer requires Premium. <a href="{url_for("pricing")}" style="color:var(--amber);">Upgrade →</a></div>' if locked else ""
    body = f"""<div class="section-label">Cashflow Analyzer</div><h1 style="margin-bottom:.25rem;">Upload Bank CSV</h1>
<p class="muted" style="margin-bottom:1.5rem;">Columns: <span class="mono">date, description, amount</span></p>
<div class="card" style="max-width:560px;">{lock_html}
  <form method="post" action="{url_for('upload_csv_post')}" enctype="multipart/form-data">
    <div class="form-group"><label>CSV File</label><input type="file" name="file" accept=".csv" {'disabled' if locked else ''} required></div>
    <button class="btn btn-primary btn-block" {'disabled' if locked else ''} type="submit">Upload + Analyze →</button>
  </form></div>"""
    return page("Upload CSV", body)

@app.post("/upload")
def upload_csv_post():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): return redirect(url_for("pricing"))
    f = request.files.get("file")
    if not f: flash("No file.", "warning"); return redirect(url_for("upload_csv"))
    raw = f.read().decode("utf-8", errors="ignore")
    reader = csv.DictReader(io.StringIO(raw))
    headers = [h.lower().strip() for h in (reader.fieldnames or [])]
    if not {"date","description","amount"}.issubset(set(headers)):
        flash("CSV must have: date, description, amount.", "warning"); return redirect(url_for("upload_csv"))
    def get(row, key):
        for k in row.keys():
            if (k or "").lower().strip() == key: return row.get(k)
        return ""
    added = 0
    for row in reader:
        ds=( get(row,"date") or "").strip(); desc=(get(row,"description") or "").strip()[:240]
        amt_s=(get(row,"amount") or "").strip().replace("$","").replace(",","")
        if not ds or not desc or not amt_s: continue
        try: d=dt.date.fromisoformat(ds); amt=float(amt_s)
        except: continue
        db.session.add(Transaction(user_id=u.id,date=d,description=desc,amount=amt,category=categorize(desc,amt))); added+=1
    db.session.commit(); flash(f"Imported {added} transactions.", "success")
    return redirect(url_for("transactions"))

@app.get("/transactions")
def transactions():
    gate = require_login()
    if gate: return gate
    u = me()
    since = dt.date.today()-dt.timedelta(days=30)
    txs   = Transaction.query.filter(Transaction.user_id==u.id).order_by(Transaction.date.desc()).limit(200).all()
    t30   = Transaction.query.filter(Transaction.user_id==u.id,Transaction.date>=since).all()
    spend = sum(-t.amount for t in t30 if t.amount<0); income = sum(t.amount for t in t30 if t.amount>0)
    by_cat: Dict[str,float] = {}
    for t in t30:
        if t.amount<0: by_cat[t.category]=by_cat.get(t.category,0.0)+(-t.amount)
    top = sorted(by_cat.items(),key=lambda x:x[1],reverse=True)[:7]
    cat_rows="".join(f'<div style="margin-bottom:.75rem;"><div style="display:flex;justify-content:space-between;margin-bottom:.3rem;"><span class="mono" style="font-size:.75rem;">{safe(cat)}</span><span class="mono" style="font-size:.75rem;color:var(--muted);">{money(val)}</span></div><div class="bar-wrap"><div class="bar {"bar-amber" if cat in ("Restaurants","Subscriptions") else "bar-green"}" style="width:{min(100,int(val/spend*100)) if spend>0 else 0}%;"></div></div></div>' for cat,val in top)
    tx_rows="".join(f'<tr><td class="muted">{t.date.isoformat()}</td><td>{safe(t.description)}</td><td><span class="badge badge-muted">{safe(t.category)}</span></td><td class="text-right {"green" if t.amount>0 else "red"}">{t.amount:+,.2f}</td></tr>' for t in txs)
    body=f"""<div class="section-label">Cashflow</div><h1 style="margin-bottom:1.5rem;">Transactions</h1>
<div class="grid" style="grid-template-columns:320px 1fr;gap:1rem;align-items:start;">
  <div class="card"><div class="section-label">Last 30 Days</div>
    <div class="grid grid-2" style="gap:.75rem;margin-bottom:1rem;">
      <div class="stat-box"><div class="val green">{money(income)}</div><div class="lbl">Income</div></div>
      <div class="stat-box"><div class="val red">{money(spend)}</div><div class="lbl">Spent</div></div>
    </div>
    <div class="section-label">By Category</div>{cat_rows or '<p class="muted" style="font-size:.82rem;">No data yet.</p>'}
    <hr><a class="btn btn-block" href="{url_for('upload_csv')}">Upload CSV →</a>
  </div>
  <div class="card"><div style="overflow-x:auto;"><table>
    <thead><tr><th>Date</th><th>Description</th><th>Category</th><th class="text-right">Amount</th></tr></thead>
    <tbody>{tx_rows or "<tr><td colspan=4 class=muted>No transactions yet.</td></tr>"}</tbody>
  </table></div></div>
</div>"""
    return page("Transactions", body)

# ── Debt Engine ───────────────────────────────────────────────────────────────

@app.get("/debt")
def debt():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): flash("Premium required for Debt Engine.", "warning"); return redirect(url_for("pricing"))
    debts = Debt.query.filter_by(user_id=u.id).order_by(Debt.apr.desc()).all()
    total_bal=sum(d.balance for d in debts); total_min=sum(d.minimum_payment for d in debts)
    debt_rows="".join(f'<tr><td>{safe(d.name)}</td><td class="text-right">{money(d.balance)}</td><td class="text-right amber">{d.apr:.2f}%</td><td class="text-right">{money(d.minimum_payment)}</td><td><form method="post" action="{url_for("debt_delete",did=d.id)}" style="display:inline;"><button class="btn btn-sm btn-danger" type="submit">×</button></form></td></tr>' for d in debts)
    body=f"""<div class="section-label">Debt Destruction Engine</div><h1 style="margin-bottom:1.5rem;">Debt Payoff Planner</h1>
<div class="grid grid-2" style="gap:1rem;margin-bottom:1rem;">
  <div class="stat-box"><div class="val red">{money(total_bal)}</div><div class="lbl">Total Balance</div></div>
  <div class="stat-box"><div class="val">{money(total_min)}</div><div class="lbl">Total Minimums/mo</div></div>
</div>
<div class="grid grid-2" style="gap:1rem;">
  <div>
    <div class="card" style="margin-bottom:1rem;"><div class="section-label">Add Debt</div>
      <form method="post" action="{url_for('debt_add')}">
        <div class="grid grid-2" style="gap:.75rem;">
          <div class="form-group"><label>Name</label><input type="text" name="name" required placeholder="Chase Visa"></div>
          <div class="form-group"><label>Balance ($)</label><input type="number" name="balance" step="1" min="0" required></div>
          <div class="form-group"><label>APR (%)</label><input type="number" name="apr" step="0.01" min="0" required></div>
          <div class="form-group"><label>Min Payment ($)</label><input type="number" name="minpay" step="1" min="0" required></div>
        </div>
        <button class="btn btn-primary btn-block" type="submit">Add Debt →</button>
      </form>
    </div>
    <div class="card"><div class="section-label">Generate Plan</div>
      <form method="post" action="{url_for('debt_plan')}">
        <div class="form-group"><label>Method</label><select name="method"><option value="avalanche">Avalanche (highest APR first)</option><option value="snowball">Snowball (smallest balance first)</option></select></div>
        <div class="form-group"><label>Extra per month ($)</label><input type="number" name="extra" step="1" min="0" value="100"></div>
        <button class="btn btn-primary btn-block" type="submit">Generate Schedule →</button>
      </form>
    </div>
  </div>
  <div class="card"><div class="section-label">Your Debts ({len(debts)})</div>
    <div style="overflow-x:auto;"><table><thead><tr><th>Name</th><th class="text-right">Balance</th><th class="text-right">APR</th><th class="text-right">Min</th><th></th></tr></thead>
    <tbody>{debt_rows or "<tr><td colspan=5 class=muted>No debts added yet.</td></tr>"}</tbody></table></div>
  </div>
</div>"""
    return page("Debt Engine", body)

@app.post("/debt/add")
def debt_add():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): return redirect(url_for("pricing"))
    try:
        name=( request.form.get("name") or "").strip()[:120]
        bal=float(request.form.get("balance") or 0.0); apr=float(request.form.get("apr") or 0.0)
        mp=float(request.form.get("minpay") or 0.0)
        if not name: raise ValueError
    except: flash("Invalid values.", "warning"); return redirect(url_for("debt"))
    db.session.add(Debt(user_id=u.id,name=name,balance=max(0,bal),apr=max(0,apr),minimum_payment=max(0,mp)))
    db.session.commit(); flash(f"Added: {name}.", "success"); return redirect(url_for("debt"))

@app.post("/debt/<int:did>/delete")
def debt_delete(did):
    gate = require_login()
    if gate: return gate
    u = me(); d=Debt.query.filter_by(id=did,user_id=u.id).first_or_404()
    db.session.delete(d); db.session.commit(); flash("Removed.", "success"); return redirect(url_for("debt"))

@app.post("/debt/plan")
def debt_plan():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): return redirect(url_for("pricing"))
    method=request.form.get("method") or "avalanche"
    try: extra=float(request.form.get("extra") or 0.0)
    except: extra=0.0
    debts=Debt.query.filter_by(user_id=u.id).all()
    months,total_interest,timeline=payoff_schedule(debts,extra,method)
    tr="".join(f'<tr><td class="muted">{m["month"]}</td><td class="amber">{money(m["interest"])}</td><td>{money(sum(p for _,p in m["payments"]))}</td><td class="{"red" if m["remaining"]>0 else "green"}">{money(m["remaining"])}</td></tr>' for m in timeline[:24])
    finish_yr=months//12; finish_mo=months%12
    body=f"""<div class="section-label">Debt Payoff Schedule</div><h1 style="margin-bottom:1.5rem;">{safe(method.title())} + {money(extra)}/mo extra</h1>
<div class="grid grid-4" style="gap:1rem;margin-bottom:1.5rem;">
  <div class="stat-box"><div class="val green">{months}</div><div class="lbl">Months to Zero</div></div>
  <div class="stat-box"><div class="val">{finish_yr}y {finish_mo}m</div><div class="lbl">Time Frame</div></div>
  <div class="stat-box"><div class="val red">{money(total_interest)}</div><div class="lbl">Total Interest</div></div>
  <div class="stat-box"><div class="val">{money(extra)}</div><div class="lbl">Extra/Month</div></div>
</div>
<div class="card"><div class="section-label">Month-by-Month (first 24)</div>
  <div style="overflow-x:auto;"><table><thead><tr><th>Month</th><th>Interest</th><th>Payments</th><th>Remaining</th></tr></thead>
  <tbody>{tr or "<tr><td colspan=4 class=muted>Add debts first.</td></tr>"}</tbody></table></div>
</div>
<div style="display:flex;gap:.75rem;margin-top:1rem;">
  <a class="btn" href="{url_for('debt')}">← Back</a>
  <a class="btn btn-primary" href="{url_for('dashboard')}">Dashboard</a>
</div>"""
    return page("Debt Plan", body)

# ── Retirement ────────────────────────────────────────────────────────────────

@app.get("/retirement")
def retirement():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): flash("Premium required.", "warning"); return redirect(url_for("pricing"))
    body=f"""<div class="section-label">Retirement Projection</div><h1 style="margin-bottom:.25rem;">Future Value Calculator</h1>
<p class="muted" style="margin-bottom:1.5rem;">Education only — not a guarantee.</p>
<div class="card" style="max-width:520px;"><form method="post" action="{url_for('retirement_post')}">
  <div class="grid grid-2" style="gap:.75rem;">
    <div class="form-group"><label>Current Age</label><input type="number" name="age" min="10" max="100" value="30" required></div>
    <div class="form-group"><label>Retire Age</label><input type="number" name="retire" min="20" max="110" value="65" required></div>
    <div class="form-group"><label>Monthly Invest ($)</label><input type="number" name="monthly" min="0" step="1" value="250" required></div>
    <div class="form-group"><label>Annual Return (%)</label><input type="number" name="r" min="0" step="0.1" value="7.0" required></div>
  </div>
  <button class="btn btn-primary btn-block" type="submit">Calculate →</button>
</form></div>"""
    return page("Retirement", body)

@app.post("/retirement")
def retirement_post():
    gate = require_login()
    if gate: return gate
    u = me()
    if not premium_gate(u): return redirect(url_for("pricing"))
    try:
        age=int(request.form.get("age") or 0); retire=int(request.form.get("retire") or 0)
        monthly=float(request.form.get("monthly") or 0.0); r=float(request.form.get("r") or 0.0)
    except: flash("Invalid inputs.", "warning"); return redirect(url_for("retirement"))
    proj=retirement_projection(age,retire,monthly,r)
    milestones="".join(f'<tr><td class="muted">{age+yr}</td><td class="muted">{yr}y</td><td class="green">{money(future_value(monthly,yr,r))}</td><td>{money((future_value(monthly,yr,r)*0.04)/12)}/mo</td></tr>' for yr in range(5,proj["years"]+1,5))
    body=f"""<div class="section-label">Retirement Result</div><h1 style="margin-bottom:1.5rem;">{money(monthly)}/mo for {proj["years"]} years @ {r}%</h1>
<div class="grid grid-3" style="gap:1rem;margin-bottom:1.5rem;">
  <div class="stat-box"><div class="val green">{money(proj["fv"])}</div><div class="lbl">Projected Value</div></div>
  <div class="stat-box"><div class="val">{money(proj["annual_income"])}</div><div class="lbl">Annual Income (4%)</div></div>
  <div class="stat-box"><div class="val">{money(proj["monthly_income"])}</div><div class="lbl">Monthly Income</div></div>
</div>
<div class="card"><div class="section-label">Milestones</div><table>
  <thead><tr><th>Age</th><th>Years</th><th>Portfolio</th><th>Income Est.</th></tr></thead>
  <tbody>{milestones or "<tr><td colspan=4 class=muted>Increase years to see milestones.</td></tr>"}</tbody>
</table></div>
<div style="display:flex;gap:.75rem;margin-top:1rem;">
  <a class="btn" href="{url_for('retirement')}">← Recalculate</a>
  <a class="btn btn-primary" href="{url_for('dashboard')}">Dashboard</a>
</div>"""
    return page("Projection", body)

# ── Community ─────────────────────────────────────────────────────────────────

@app.get("/community")
def community():
    gate = require_login()
    if gate: return gate
    u = me()
    posts=CommunityPost.query.order_by(CommunityPost.created_at.desc()).limit(50).all()
    post_html="".join(f'<div class="card card-sm" style="margin-bottom:.75rem;"><div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:.4rem;"><strong class="mono" style="font-size:.82rem;color:var(--green);">@{safe(db.session.get(User,p.user_id).handle if db.session.get(User,p.user_id) else "unknown")}</strong><span class="muted mono" style="font-size:.72rem;">{p.created_at.strftime("%Y-%m-%d")}</span></div><div style="font-weight:600;margin-bottom:.3rem;font-size:.9rem;">{safe(p.title)}</div><p class="muted" style="font-size:.82rem;line-height:1.6;">{safe(p.body)}</p></div>' for p in posts)
    body=f"""<div class="section-label">Community</div><h1 style="margin-bottom:.25rem;">Share Your Wins</h1>
<p class="muted" style="margin-bottom:1.5rem;">Progress shared here keeps everyone motivated.</p>
<div class="grid" style="grid-template-columns:360px 1fr;gap:1rem;align-items:start;">
  <div class="card"><div class="section-label">New Post</div>
    <form method="post" action="{url_for('community_post')}">
      <div class="form-group"><label>Title</label><input type="text" name="title" maxlength="120" required placeholder="Paid off my car loan!"></div>
      <div class="form-group"><label>Post</label><textarea name="body" rows="5" maxlength="1200" required></textarea></div>
      <button class="btn btn-primary btn-block" type="submit">Post →</button>
    </form>
  </div>
  <div>{post_html or '<div class="card card-sm muted">No posts yet. Be the first.</div>'}</div>
</div>"""
    return page("Community", body)

@app.post("/community/post")
def community_post():
    gate = require_login()
    if gate: return gate
    u = me(); title=(request.form.get("title") or "").strip()[:120]; body_=(request.form.get("body") or "").strip()
    if not title or not body_: flash("Title and body required.", "warning"); return redirect(url_for("community"))
    db.session.add(CommunityPost(user_id=u.id,title=title,body=body_[:1200]))
    db.session.commit(); flash("Posted.", "success"); return redirect(url_for("community"))

# ── Organization ──────────────────────────────────────────────────────────────

def gen_invite_code(name):
    base="".join([c for c in name.upper() if c.isalnum()])[:8] or "ORG"
    return f"{base}-{int(dt.datetime.utcnow().timestamp())}"

@app.get("/org")
def org():
    gate = require_login()
    if gate: return gate
    u = me()
    org_info=""
    if u.org_id:
        o=db.session.get(Organization,u.org_id); seat_used=User.query.filter_by(org_id=u.org_id).count()
        org_info=f'<div class="card" style="margin-bottom:1rem;border-color:var(--green);"><div class="section-label">Current Organization</div><div class="mono" style="font-size:1.1rem;margin-bottom:.5rem;">{safe(o.name)}</div><div class="grid grid-3" style="gap:.75rem;margin-bottom:.75rem;"><div class="stat-box"><div class="val">{seat_used}</div><div class="lbl">Seats Used</div></div><div class="stat-box"><div class="val">{o.seat_limit}</div><div class="lbl">Seat Limit</div></div><div class="stat-box"><div class="val"><span class="badge badge-{"green" if u.org_role=="admin" else "muted"}">{u.org_role}</span></div><div class="lbl">Your Role</div></div></div><p class="muted" style="font-size:.8rem;">Invite code: <span class="mono green">{o.invite_code}</span></p></div>'
    body=f"""<div class="section-label">Organization</div><h1 style="margin-bottom:.25rem;">B2B Portal</h1>
<p class="muted" style="margin-bottom:1.5rem;">License seats to employers, credit unions, schools.</p>
{org_info}
<div class="grid grid-2" style="gap:1rem;">
  <div class="card"><div class="section-label">Create Organization</div>
    <form method="post" action="{url_for('org_create')}">
      <div class="form-group"><label>Organization Name</label><input type="text" name="name" required placeholder="Acme Credit Union"></div>
      <div class="form-group"><label>Seat Limit</label><input type="number" name="seats" min="1" max="5000" value="25"></div>
      <button class="btn btn-primary btn-block" type="submit">Create →</button>
    </form>
  </div>
  <div class="card"><div class="section-label">Join Organization</div>
    <form method="post" action="{url_for('org_join')}">
      <div class="form-group"><label>Invite Code</label><input type="text" name="code" class="mono" required></div>
      <button class="btn btn-block" type="submit">Join →</button>
    </form>
    <hr><div class="section-label">Issue Premium Keys</div>
    <form method="post" action="{url_for('issue_key')}">
      <div class="form-group"><label>Master Key</label><input type="password" name="master" required></div>
      <div class="form-group"><label>How Many?</label><input type="number" name="n" min="1" max="50" value="5"></div>
      <button class="btn btn-block" type="submit">Generate Keys →</button>
    </form>
  </div>
</div>"""
    return page("Organization", body)

@app.post("/org/create")
def org_create():
    gate = require_login()
    if gate: return gate
    u = me(); name=(request.form.get("name") or "").strip()[:120]
    try: seats=max(1,min(5000,int(request.form.get("seats") or 25)))
    except: seats=25
    if not name: flash("Org name required.", "warning"); return redirect(url_for("org"))
    o=Organization(name=name,seat_limit=seats,invite_code=gen_invite_code(name))
    db.session.add(o); db.session.commit(); u.org_id=o.id; u.org_role="admin"; db.session.commit()
    flash(f"Organization '{name}' created.", "success"); return redirect(url_for("org"))

@app.post("/org/join")
def org_join():
    gate = require_login()
    if gate: return gate
    u = me(); code=(request.form.get("code") or "").strip()
    o=Organization.query.filter_by(invite_code=code).first()
    if not o: flash("Invalid invite code.", "warning"); return redirect(url_for("org"))
    if User.query.filter_by(org_id=o.id).count()>=o.seat_limit: flash("Organization is full.", "warning"); return redirect(url_for("org"))
    u.org_id=o.id; u.org_role="member"; db.session.commit()
    flash(f"Joined {o.name}. Premium is now active.", "success"); return redirect(url_for("dashboard"))

@app.post("/org/issue_key")
def issue_key():
    gate = require_login()
    if gate: return gate
    master=(request.form.get("master") or "").strip()
    if master!=PREMIUM_MASTER_KEY: flash("Invalid master key.", "warning"); return redirect(url_for("org"))
    try: n=max(1,min(50,int(request.form.get("n") or 1)))
    except: n=1
    keys=[]
    for i in range(n):
        k=f"FFOS-{int(dt.datetime.utcnow().timestamp())}-{i+1:03d}"
        db.session.add(PremiumKey(key=k)); keys.append(k)
    db.session.commit()
    body=f'<div class="section-label">Generated Keys</div><h1 style="margin-bottom:1rem;">Copy These Now</h1><div class="card"><pre class="mono" style="white-space:pre-wrap;color:var(--green);font-size:.85rem;line-height:2;">{chr(10).join(keys)}</pre></div><div style="margin-top:1rem;"><a class="btn" href="{url_for("org")}">← Back</a></div>'
    return page("Keys Generated", body)

# ── Admin ─────────────────────────────────────────────────────────────────────

def require_admin():
    u = me()
    if not u: flash("Login required.", "warning"); return redirect(url_for("login"))
    if u.org_role != "admin": flash("Admin access required.", "warning"); return redirect(url_for("dashboard"))
    return None

@app.get("/admin/lessons")
def admin_lessons():
    gate = require_admin()
    if gate: return gate
    lessons=Lesson.query.order_by(Lesson.module,Lesson.order_index).all()
    rows="".join(f'<tr><td>{l.order_index}</td><td>{safe(l.module)}</td><td><a href="{url_for("admin_lesson_edit",lid=l.id)}" style="color:var(--text);text-decoration:none;">{safe(l.title)}</a></td><td><span class="badge badge-muted">{safe(l.level)}</span></td><td>{len(l.questions)}</td><td><span class="badge {"badge-green" if l.is_published else "badge-muted"}">{"Published" if l.is_published else "Draft"}</span></td><td style="display:flex;gap:.4rem;"><a class="btn btn-sm" href="{url_for("admin_lesson_edit",lid=l.id)}">Edit</a><form method="post" action="{url_for("admin_lesson_toggle",lid=l.id)}" style="display:inline;"><button class="btn btn-sm {"btn-amber" if l.is_published else "btn-primary"}" type="submit">{"Unpublish" if l.is_published else "Publish"}</button></form><form method="post" action="{url_for("admin_lesson_delete",lid=l.id)}" style="display:inline;"><button class="btn btn-sm btn-danger" type="submit" onclick="return confirm(\'Delete?\')">Delete</button></form></td></tr>' for l in lessons)
    body=f'<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:1.5rem;flex-wrap:wrap;gap:.75rem;"><div><div class="section-label">Admin Panel</div><h1>Lesson Manager</h1></div><a class="btn btn-primary" href="{url_for("admin_lesson_new")}">+ New Lesson</a></div><div class="card"><div style="overflow-x:auto;"><table><thead><tr><th>#</th><th>Module</th><th>Title</th><th>Level</th><th>Quiz Qs</th><th>Status</th><th>Actions</th></tr></thead><tbody>{rows or "<tr><td colspan=7 class=muted>No lessons yet.</td></tr>"}</tbody></table></div></div>'
    return page("Admin: Lessons", body)

@app.get("/admin/lessons/new")
def admin_lesson_new():
    gate = require_admin()
    if gate: return gate
    return _lesson_form(None)

@app.get("/admin/lessons/<int:lid>/edit")
def admin_lesson_edit(lid):
    gate = require_admin()
    if gate: return gate
    lesson=db.session.get(Lesson,lid)
    if not lesson: abort(404)
    return _lesson_form(lesson)

def _lesson_form(lesson):
    is_edit    = lesson is not None
    action     = url_for("admin_lesson_save", lid=lesson.id) if is_edit else url_for("admin_lesson_create")
    title_val  = safe(lesson.title if is_edit else "")
    mod_val    = lesson.module if is_edit else "Foundations"
    level_val  = lesson.level  if is_edit else "All"
    order_val  = lesson.order_index if is_edit else 1
    content_v  = lesson.content if is_edit else ""
    page_title = ("Edit" if is_edit else "New") + " Lesson"

    module_opts = "".join(
        '<option value="{}" {}>{}</option>'.format(m, "selected" if mod_val == m else "", m)
        for m in ["Foundations", "Debt", "Investing", "Savings", "Advanced"]
    )
    level_opts = "".join(
        '<option value="{}" {}>{}</option>'.format(lv, "selected" if level_val == lv else "", lv)
        for lv in ["All", "Parent", "Teen"]
    )

    # Existing questions block
    q_section = ""
    if is_edit and lesson.questions:
        q_rows = ""
        for q in lesson.questions:
            del_url = url_for("admin_question_delete", qid=q.id)
            opt_html = ""
            for letter in ["A", "B", "C", "D"]:
                bg  = "rgba(0,232,122,.1)" if letter == q.correct.upper() else "transparent"
                col = "var(--green)"        if letter == q.correct.upper() else "var(--muted)"
                opt_html += (
                    '<div style="font-size:.78rem;padding:.2rem .4rem;border-radius:2px;'
                    'margin-bottom:.2rem;background:{};">'.format(bg) +
                    '<span class="mono" style="color:{};">{}.'.format(col, letter) +
                    '</span> {}</div>'.format(safe(getattr(q, "opt_" + letter.lower())))
                )
            q_rows += (
                '<div class="card card-sm" style="margin-bottom:.75rem;">'
                '<div style="display:flex;justify-content:space-between;align-items:flex-start;gap:.5rem;">'
                '<p style="font-family:var(--mono);font-size:.82rem;font-weight:600;margin-bottom:.5rem;flex:1;">'
                + safe(q.prompt) +
                '</p><form method="post" action="' + del_url + '" style="flex-shrink:0;">'
                '<button class="btn btn-sm btn-danger" type="submit">\xd7</button></form></div>'
                + opt_html + '</div>'
            )
        q_section = (
            '<hr><div class="section-label">Existing Questions ({})</div>'.format(len(lesson.questions))
            + q_rows
        )

    # Add question form
    add_q = ""
    if is_edit:
        add_url   = url_for("admin_question_add", lid=lesson.id)
        opt_fields = "".join(
            '<div class="form-group"><label>Option {}</label>'
            '<input type="text" name="opt_{}" maxlength="200" required></div>'.format(l, l.lower())
            for l in ["A", "B", "C", "D"]
        )
        correct_opts = "".join('<option value="{}">{}</option>'.format(l, l) for l in ["A","B","C","D"])
        add_q = (
            '<hr><div class="section-label">Add Quiz Question</div>'
            '<form method="post" action="' + add_url + '">'
            '<div class="form-group"><label>Question Prompt</label>'
            '<input type="text" name="prompt" maxlength="300" required></div>'
            '<div class="grid grid-2" style="gap:.5rem .75rem;">' + opt_fields +
            '<div class="form-group"><label>Correct</label>'
            '<select name="correct">' + correct_opts + '</select></div>'
            '<div class="form-group"><label>Explanation</label>'
            '<input type="text" name="explanation" maxlength="500"></div></div>'
            '<button class="btn btn-primary" type="submit">Add Question</button></form>'
        )

    pub_sel1 = "selected" if (not is_edit or lesson.is_published) else ""
    pub_sel2 = "selected" if (is_edit and not lesson.is_published) else ""
    submit_label = "Save Changes" if is_edit else "Create Lesson"
    admin_url    = url_for("admin_lessons")

    body = (
        '<div style="max-width:760px;margin:0 auto;">'
        '<div class="section-label">Admin Panel</div>'
        '<h1 style="margin-bottom:1.5rem;">' + page_title + '</h1>'
        '<div class="card">'
        '<form method="post" action="' + action + '">'
        '<div class="grid grid-2" style="gap:.5rem .75rem;">'
        '<div class="form-group" style="grid-column:1/-1;"><label>Title</label>'
        '<input type="text" name="title" maxlength="160" required value="' + title_val + '"></div>'
        '<div class="form-group"><label>Module</label><select name="module">' + module_opts + '</select></div>'
        '<div class="form-group"><label>Level</label><select name="level">' + level_opts + '</select></div>'
        '<div class="form-group"><label>Order Index</label>'
        '<input type="number" name="order_index" min="1" max="999" value="' + str(order_val) + '"></div>'
        '<div class="form-group"><label>Published</label><select name="is_published">'
        '<option value="1" ' + pub_sel1 + '>Yes</option>'
        '<option value="0" ' + pub_sel2 + '>No (Draft)</option></select></div>'
        '</div>'
        '<div class="form-group"><label>Content</label>'
        '<textarea name="content" rows="10" required style="font-family:var(--mono);font-size:.82rem;">'
        + safe(content_v) +
        '</textarea></div>'
        '<button class="btn btn-primary" type="submit">' + submit_label + '</button>'
        '&nbsp;<a class="btn" href="' + admin_url + '">Cancel</a>'
        '</form>'
        + q_section + add_q +
        '</div>'
        '<div style="margin-top:1rem;"><a class="btn" href="' + admin_url + '">\u2190 All Lessons</a></div>'
        '</div>'
    )
    return page(page_title, body)

@app.post("/admin/lessons/create")
def admin_lesson_create():
    gate = require_admin()
    if gate: return gate
    title=(request.form.get("title") or "").strip()[:160]; module=(request.form.get("module") or "Foundations").strip()
    level=(request.form.get("level") or "All").strip(); content=(request.form.get("content") or "").strip()
    try: order=max(1,int(request.form.get("order_index") or 1))
    except: order=1
    published=request.form.get("is_published")!="0"
    if not title or not content: flash("Title and content required.", "warning"); return redirect(url_for("admin_lesson_new"))
    lesson=Lesson(title=title,module=module,level=level,content=content,order_index=order,is_published=published)
    db.session.add(lesson); db.session.commit(); flash(f"Lesson '{title}' created.", "success")
    return redirect(url_for("admin_lesson_edit",lid=lesson.id))

@app.post("/admin/lessons/<int:lid>/save")
def admin_lesson_save(lid):
    gate = require_admin()
    if gate: return gate
    lesson=db.session.get(Lesson,lid)
    if not lesson: abort(404)
    lesson.title=(request.form.get("title") or "").strip()[:160] or lesson.title
    lesson.module=(request.form.get("module") or lesson.module).strip()
    lesson.level=(request.form.get("level") or lesson.level).strip()
    lesson.content=(request.form.get("content") or "").strip() or lesson.content
    lesson.is_published=request.form.get("is_published")!="0"
    try: lesson.order_index=max(1,int(request.form.get("order_index") or 1))
    except: pass
    db.session.commit(); flash("Saved.", "success"); return redirect(url_for("admin_lesson_edit",lid=lid))

@app.post("/admin/lessons/<int:lid>/toggle")
def admin_lesson_toggle(lid):
    gate = require_admin()
    if gate: return gate
    lesson=db.session.get(Lesson,lid)
    if not lesson: abort(404)
    lesson.is_published=not lesson.is_published; db.session.commit()
    flash(f"'{lesson.title}' {'published' if lesson.is_published else 'set to draft'}.", "success")
    return redirect(url_for("admin_lessons"))

@app.post("/admin/lessons/<int:lid>/delete")
def admin_lesson_delete(lid):
    gate = require_admin()
    if gate: return gate
    lesson=db.session.get(Lesson,lid)
    if not lesson: abort(404)
    LessonProgress.query.filter_by(lesson_id=lid).delete()
    QuizQuestion.query.filter_by(lesson_id=lid).delete()
    db.session.delete(lesson); db.session.commit(); flash("Deleted.", "success")
    return redirect(url_for("admin_lessons"))

@app.post("/admin/lessons/<int:lid>/questions/add")
def admin_question_add(lid):
    gate = require_admin()
    if gate: return gate
    lesson=db.session.get(Lesson,lid)
    if not lesson: abort(404)
    prompt=(request.form.get("prompt") or "").strip()[:300]
    a=(request.form.get("opt_a") or "").strip()[:200]; b=(request.form.get("opt_b") or "").strip()[:200]
    c=(request.form.get("opt_c") or "").strip()[:200]; d=(request.form.get("opt_d") or "").strip()[:200]
    correct=(request.form.get("correct") or "A").upper(); expl=(request.form.get("explanation") or "").strip()[:500]
    if not prompt or not all([a,b,c,d]) or correct not in "ABCD":
        flash("All fields required.", "warning"); return redirect(url_for("admin_lesson_edit",lid=lid))
    db.session.add(QuizQuestion(lesson_id=lid,prompt=prompt,a=a,b=b,c=c,d=d,correct=correct,explanation=expl))
    db.session.commit(); flash("Question added.", "success")
    return redirect(url_for("admin_lesson_edit",lid=lid))

@app.post("/admin/questions/<int:qid>/delete")
def admin_question_delete(qid):
    gate = require_admin()
    if gate: return gate
    q=db.session.get(QuizQuestion,qid)
    if not q: abort(404)
    lid=q.lesson_id; db.session.delete(q); db.session.commit(); flash("Removed.", "success")
    return redirect(url_for("admin_lesson_edit",lid=lid))

# ── Budget ────────────────────────────────────────────────────────────────────

DEFAULT_BUDGET_CATEGORIES=["Rent/Mortgage","Utilities","Groceries","Transport","Insurance","Subscriptions","Restaurants","Clothing","Healthcare","Debt Payments","Savings","Investing","Other"]

@app.get("/budget")
def budget():
    gate = require_login()
    if gate: return gate
    u = me()
    this_month=dt.date.today().strftime("%Y-%m"); month_start=dt.date.today().replace(day=1)
    plan=BudgetPlan.query.filter_by(user_id=u.id,month=this_month).first()
    txs=Transaction.query.filter(Transaction.user_id==u.id,Transaction.date>=month_start).all()
    actuals: Dict[str,float]={}
    for t in txs:
        if t.amount<0: actuals[t.category]=actuals.get(t.category,0.0)+(-t.amount)
    if plan:
        planned_total=sum(c.planned_amount for c in plan.categories); remaining=(plan.total_income_planned or 0.0)-planned_total
        rows="".join(f'<tr><td>{safe(c.category)}</td><td class="text-right">{money(c.planned_amount)}</td><td class="text-right">{money(actuals.get(c.category,0.0))}</td><td class="text-right {"green" if c.planned_amount-actuals.get(c.category,0.0)>=0 else "red"}">{money(c.planned_amount-actuals.get(c.category,0.0))}</td></tr>' for c in sorted(plan.categories,key=lambda x:x.planned_amount,reverse=True))
        plan_html=f'<div class="grid grid-3" style="gap:1rem;margin-bottom:1rem;"><div class="stat-box"><div class="val">{money(plan.total_income_planned)}</div><div class="lbl">Income Planned</div></div><div class="stat-box"><div class="val">{money(planned_total)}</div><div class="lbl">Budgeted</div></div><div class="stat-box"><div class="val {"green" if remaining>=0 else "red"}">{money(remaining)}</div><div class="lbl">{"Unallocated" if remaining>=0 else "Over Budget"}</div></div></div><div class="card"><div class="section-label">Categories — {this_month}</div><div style="overflow-x:auto;"><table><thead><tr><th>Category</th><th class="text-right">Budget</th><th class="text-right">Actual</th><th class="text-right">Diff</th></tr></thead><tbody>{rows}</tbody></table></div><hr><a class="btn btn-sm" href="{url_for("budget_edit",month=this_month)}">Edit Budget →</a></div>'
    else:
        plan_html=f'<div class="card" style="text-align:center;padding:2.5rem;"><p class="muted" style="margin-bottom:1rem;">No budget plan for {this_month} yet.</p><a class="btn btn-primary" href="{url_for("budget_edit",month=this_month)}">Create Budget →</a></div>'
    body=f'<div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:1.5rem;flex-wrap:wrap;gap:.75rem;"><div><div class="section-label">Budget Planner</div><h1>{this_month}</h1></div><a class="btn btn-sm btn-primary" href="{url_for("budget_edit",month=this_month)}">Edit Budget</a></div>{plan_html}'
    return page("Budget Planner", body)

@app.get("/budget/<month>/edit")
def budget_edit(month):
    gate = require_login()
    if gate: return gate
    u = me()
    try: dt.datetime.strptime(month,"%Y-%m")
    except: abort(400)
    plan=BudgetPlan.query.filter_by(user_id=u.id,month=month).first()
    existing={c.category:c.planned_amount for c in plan.categories} if plan else {}
    income_planned=plan.total_income_planned if plan else 0.0; notes=plan.notes if plan else ""
    cat_inputs="".join(f'<div class="form-group"><label>{safe(cat)}</label><input type="number" name="cat_{cat.replace("/","_").replace(" ","_")}" step="1" min="0" value="{existing.get(cat,0.0):.0f}"></div>' for cat in DEFAULT_BUDGET_CATEGORIES)
    body=f'<div style="max-width:680px;margin:0 auto;"><div class="section-label">Budget Planner</div><h1 style="margin-bottom:1.5rem;">Edit Budget — {month}</h1><div class="card"><form method="post" action="{url_for("budget_save",month=month)}"><div class="form-group"><label>Total Income This Month ($)</label><input type="number" name="income" step="1" min="0" value="{income_planned:.0f}" required></div><hr><div class="section-label">Category Budgets</div><div class="grid grid-2" style="gap:.5rem .75rem;">{cat_inputs}</div><hr><div class="form-group"><label>Notes</label><input type="text" name="notes" maxlength="240" value="{safe(notes)}"></div><button class="btn btn-primary btn-block" type="submit">Save Budget →</button></form></div><div style="margin-top:1rem;"><a class="btn" href="{url_for("budget")}">← Back</a></div></div>'
    return page(f"Edit Budget {month}", body)

@app.post("/budget/<month>/save")
def budget_save(month):
    gate = require_login()
    if gate: return gate
    u = me()
    try: dt.datetime.strptime(month,"%Y-%m"); income=max(0.0,float(request.form.get("income") or 0.0))
    except: flash("Invalid input.", "warning"); return redirect(url_for("budget"))
    notes=(request.form.get("notes") or "")[:240]
    plan=BudgetPlan.query.filter_by(user_id=u.id,month=month).first()
    if not plan: plan=BudgetPlan(user_id=u.id,month=month); db.session.add(plan); db.session.flush()
    else: BudgetCategoryPlan.query.filter_by(budget_id=plan.id).delete()
    plan.total_income_planned=income; plan.notes=notes
    for cat in DEFAULT_BUDGET_CATEGORIES:
        key="cat_"+cat.replace("/","_").replace(" ","_")
        try: amt=max(0.0,float(request.form.get(key) or 0.0))
        except: amt=0.0
        if amt>0: db.session.add(BudgetCategoryPlan(budget_id=plan.id,category=cat,planned_amount=amt))
    db.session.commit(); flash(f"Budget saved for {month}.", "success"); return redirect(url_for("budget"))

# ── Legal & Static Pages ──────────────────────────────────────────────────────

import datetime

def _legal_page(title, html_content):
    """Render a legal page with date substitutions."""
    today = datetime.date.today()
    effective = "January 1, 2025"
    updated   = today.strftime("%B %d, %Y")
    year      = today.year
    content   = html_content.replace("{effective_date}", effective)\
                             .replace("{updated_date}", updated)\
                             .replace("{year}", str(year))
    return content

PRIVACY_HTML = """<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Privacy Policy | LegacyLift</title>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600&family=DM+Sans:wght@400;500&display=swap" rel="stylesheet"/>
<style>*{box-sizing:border-box;margin:0;padding:0}
:root{--bg:#09090b;--s1:#111114;--bord:rgba(255,255,255,.08);--text:#f0eeea;--muted:#6b6b78;--gold:#c8a84b;--lift:#7c6af5;--sans:'DM Sans',sans-serif;--serif:'Cormorant Garamond',serif}
body{font-family:var(--sans);background:var(--bg);color:var(--text);line-height:1.7}
nav{background:var(--s1);border-bottom:1px solid var(--bord);padding:1rem 2rem;display:flex;align-items:center;justify-content:space-between}
.brand{font-family:var(--serif);font-size:1.2rem;font-weight:600;background:linear-gradient(135deg,#a89cf8,var(--gold));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;text-decoration:none}
.back{font-size:.8rem;color:var(--muted);text-decoration:none}.back:hover{color:var(--text)}
main{max-width:760px;margin:0 auto;padding:3rem 2rem}
h1{font-family:var(--serif);font-size:2.4rem;font-weight:600;color:var(--text);margin-bottom:.4rem}
.meta{font-size:.8rem;color:var(--muted);margin-bottom:2.5rem;padding-bottom:1.5rem;border-bottom:1px solid var(--bord)}
h2{font-family:var(--serif);font-size:1.3rem;font-weight:600;color:var(--text);margin:2rem 0 .75rem}
p{font-size:.9rem;color:var(--muted);margin-bottom:1rem;line-height:1.8}
ul{padding-left:1.5rem;margin-bottom:1rem}li{font-size:.9rem;color:var(--muted);line-height:1.8;margin-bottom:.25rem}
a{color:var(--gold);text-decoration:none}a:hover{text-decoration:underline}
.highlight{background:rgba(124,106,245,.08);border:1px solid rgba(124,106,245,.2);border-radius:10px;padding:1.1rem 1.25rem;margin:1.5rem 0}
.highlight p{margin:0;color:var(--text);font-size:.85rem}
footer{text-align:center;padding:2rem;border-top:1px solid var(--bord);font-size:.78rem;color:var(--muted)}</style></head>
<body>
<nav><a class="brand" href="/">LegacyLift</a><a class="back" href="/">← Back</a></nav>
<main>
<h1>Privacy Policy</h1>
<div class="meta"><strong>LegacyLift LLC</strong> &nbsp;·&nbsp; Effective: {effective_date} &nbsp;·&nbsp; Updated: {updated_date}</div>
<div class="highlight"><p><strong>The short version:</strong> We collect only what we need, never sell your data, and you can delete everything by emailing <a href="mailto:support@legacylift.app">support@legacylift.app</a>.</p></div>
<h2>1. Who We Are</h2><p>LegacyLift LLC operates the LegacyLift financial education platform at legacylift.app. We are not a licensed financial advisor, broker, or investment firm. Contact: <a href="mailto:support@legacylift.app">support@legacylift.app</a></p>
<h2>2. Information We Collect</h2><p>We collect information you provide (email, password, financial profile data, CSV uploads, community posts, payment info via Stripe) and information collected automatically (IP address, browser type, feature usage, lesson progress).</p>
<h2>3. How We Use It</h2><p>To provide the platform, calculate your Freedom Score, process payments via Stripe, send transactional and opt-in marketing emails, and respond to support requests. We do <strong>not</strong> sell your data.</p>
<h2>4. Financial Data</h2><p>Your financial data is encrypted at rest, never shared with third parties for marketing, never used for lending decisions, and deletable at your request. CSV uploads are processed and discarded — we do not store your original bank files or require bank login credentials.</p>
<h2>5. Third-Party Sharing</h2><p>We share data only with Stripe (payments), Render (hosting), email service providers (for emails you've consented to), and law enforcement when required by valid legal process.</p>
<h2>6. Your Rights</h2><p>You may access, correct, delete, or export your data at any time. Email <a href="mailto:support@legacylift.app">support@legacylift.app</a> with subject "Privacy Request."</p>
<h2>7. Children's Privacy</h2><p>The children's curriculum is designed for use under parent/guardian supervision. We do not knowingly collect data directly from children under 13. Contact us immediately if you believe this has occurred.</p>
<h2>8. Data Retention</h2><p>We retain account data while your account is active. Deletion requests are processed within 30 days. Billing records are retained 7 years for tax compliance.</p>
<h2>9. Security</h2><p>We use HTTPS/TLS, encrypted storage, and secure password hashing. No method is 100% secure, but we implement industry best practices.</p>
<h2>10. Changes</h2><p>Material changes will be communicated by email. Continued use after changes constitutes acceptance.</p>
<h2>11. Contact</h2><p>Email: <a href="mailto:support@legacylift.app">support@legacylift.app</a> — Subject: "Privacy Policy Question"</p>
</main>
<footer>&copy; {year} LegacyLift LLC &nbsp;·&nbsp; <a href="/privacy">Privacy</a> &nbsp;·&nbsp; <a href="/terms">Terms</a> &nbsp;·&nbsp; <a href="/disclaimer">Disclaimer</a> &nbsp;·&nbsp; Education only — not financial advice</footer>
</body></html>"""

TERMS_HTML = """<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Terms of Service | LegacyLift</title>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600&family=DM+Sans:wght@400;500&display=swap" rel="stylesheet"/>
<style>*{box-sizing:border-box;margin:0;padding:0}:root{--bg:#09090b;--s1:#111114;--bord:rgba(255,255,255,.08);--text:#f0eeea;--muted:#6b6b78;--gold:#c8a84b;--sans:'DM Sans',sans-serif;--serif:'Cormorant Garamond',serif}body{font-family:var(--sans);background:var(--bg);color:var(--text);line-height:1.7}nav{background:var(--s1);border-bottom:1px solid var(--bord);padding:1rem 2rem;display:flex;align-items:center;justify-content:space-between}.brand{font-family:var(--serif);font-size:1.2rem;font-weight:600;background:linear-gradient(135deg,#a89cf8,var(--gold));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;text-decoration:none}.back{font-size:.8rem;color:var(--muted);text-decoration:none}.back:hover{color:var(--text)}main{max-width:760px;margin:0 auto;padding:3rem 2rem}h1{font-family:var(--serif);font-size:2.4rem;font-weight:600;color:var(--text);margin-bottom:.4rem}.meta{font-size:.8rem;color:var(--muted);margin-bottom:2.5rem;padding-bottom:1.5rem;border-bottom:1px solid var(--bord)}h2{font-family:var(--serif);font-size:1.3rem;font-weight:600;color:var(--text);margin:2rem 0 .75rem}p{font-size:.9rem;color:var(--muted);margin-bottom:1rem;line-height:1.8}ul{padding-left:1.5rem;margin-bottom:1rem}li{font-size:.9rem;color:var(--muted);line-height:1.8;margin-bottom:.25rem}a{color:var(--gold);text-decoration:none}a:hover{text-decoration:underline}.warning{background:rgba(240,85,85,.07);border:1px solid rgba(240,85,85,.2);border-radius:10px;padding:1.1rem 1.25rem;margin:1.5rem 0}.warning p{margin:0;color:#f08888;font-size:.85rem;font-weight:500}footer{text-align:center;padding:2rem;border-top:1px solid var(--bord);font-size:.78rem;color:var(--muted)}</style></head>
<body><nav><a class="brand" href="/">LegacyLift</a><a class="back" href="/">← Back</a></nav>
<main><h1>Terms of Service</h1><div class="meta"><strong>LegacyLift LLC</strong> &nbsp;·&nbsp; Effective: {effective_date} &nbsp;·&nbsp; Updated: {updated_date}</div>
<div class="warning"><p>⚠ LegacyLift is an educational platform. We are NOT licensed financial advisors. Nothing here constitutes financial advice. See Section 4.</p></div>
<h2>1. Acceptance</h2><p>By using LegacyLift, you agree to these Terms. If you disagree, do not use the platform.</p>
<h2>2. Eligibility</h2><p>You must be 18+ to create an account. The children's curriculum is for use under parent or guardian supervision.</p>
<h2>3. Billing & Refunds</h2><p><strong>Free Tier:</strong> No cost, limited features. <strong>Monthly ($9.99/mo):</strong> Cancel anytime; access continues through the billing period. <strong>Lifetime ($49):</strong> One-time payment, permanent access. <strong>Refunds:</strong> 30 days on monthly, 7 days on lifetime. Email <a href="mailto:support@legacylift.app">support@legacylift.app</a> to request.</p>
<h2>4. Not Financial Advice</h2><div class="warning"><p>LegacyLift is NOT a registered investment advisor, broker-dealer, or licensed financial professional. All content — including Freedom Score calculations, debt projections, retirement estimates, and lessons — is for educational and informational purposes only. Consult a qualified financial professional before making financial decisions. We assume no liability for financial decisions made based on Platform content.</p></div>
<h2>5. Acceptable Use</h2><p>You agree not to use the Platform unlawfully, scrape content commercially, share account credentials, upload malicious code, or impersonate others.</p>
<h2>6. Intellectual Property</h2><p>All Platform content is owned by LegacyLift LLC. You receive a limited personal license to use the Platform. No commercial reproduction without written permission.</p>
<h2>7. Disclaimer of Warranties</h2><p>THE PLATFORM IS PROVIDED "AS IS" WITHOUT WARRANTIES OF ANY KIND.</p>
<h2>8. Limitation of Liability</h2><p>LEGACYLIFT LLC SHALL NOT BE LIABLE FOR ANY INDIRECT, INCIDENTAL, OR CONSEQUENTIAL DAMAGES. OUR TOTAL LIABILITY SHALL NOT EXCEED AMOUNTS YOU PAID IN THE PRECEDING 12 MONTHS.</p>
<h2>9. Termination</h2><p>We may suspend or terminate accounts for Terms violations. You may delete your account at any time.</p>
<h2>10. Governing Law</h2><p>These Terms are governed by U.S. law. Disputes resolved through binding arbitration.</p>
<h2>11. Contact</h2><p><a href="mailto:support@legacylift.app">support@legacylift.app</a></p>
</main>
<footer>&copy; {year} LegacyLift LLC &nbsp;·&nbsp; <a href="/privacy">Privacy</a> &nbsp;·&nbsp; <a href="/terms">Terms</a> &nbsp;·&nbsp; <a href="/disclaimer">Disclaimer</a> &nbsp;·&nbsp; Education only — not financial advice</footer>
</body></html>"""

DISCLAIMER_HTML = """<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Financial Disclaimer | LegacyLift</title>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@400;600&family=DM+Sans:wght@400;500&display=swap" rel="stylesheet"/>
<style>*{box-sizing:border-box;margin:0;padding:0}:root{--bg:#09090b;--s1:#111114;--bord:rgba(255,255,255,.08);--text:#f0eeea;--muted:#6b6b78;--gold:#c8a84b;--sans:'DM Sans',sans-serif;--serif:'Cormorant Garamond',serif}body{font-family:var(--sans);background:var(--bg);color:var(--text);line-height:1.7}nav{background:var(--s1);border-bottom:1px solid var(--bord);padding:1rem 2rem;display:flex;align-items:center;justify-content:space-between}.brand{font-family:var(--serif);font-size:1.2rem;font-weight:600;background:linear-gradient(135deg,#a89cf8,var(--gold));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;text-decoration:none}.back{font-size:.8rem;color:var(--muted);text-decoration:none}.back:hover{color:var(--text)}main{max-width:760px;margin:0 auto;padding:3rem 2rem}h1{font-family:var(--serif);font-size:2.4rem;font-weight:600;color:var(--text);margin-bottom:.4rem}.meta{font-size:.8rem;color:var(--muted);margin-bottom:2.5rem;padding-bottom:1.5rem;border-bottom:1px solid var(--bord)}h2{font-family:var(--serif);font-size:1.3rem;font-weight:600;color:var(--text);margin:2rem 0 .75rem}p{font-size:.9rem;color:var(--muted);margin-bottom:1rem;line-height:1.8}a{color:var(--gold);text-decoration:none}a:hover{text-decoration:underline}.big-warning{background:rgba(240,85,85,.08);border:2px solid rgba(240,85,85,.3);border-radius:14px;padding:1.75rem;margin:1.5rem 0;text-align:center}.big-warning h2{margin:0 0 .75rem;font-size:1.5rem;color:#f08888}.big-warning p{margin:0;color:#e8aaaa;font-size:.88rem;line-height:1.75}footer{text-align:center;padding:2rem;border-top:1px solid var(--bord);font-size:.78rem;color:var(--muted)}</style></head>
<body><nav><a class="brand" href="/">LegacyLift</a><a class="back" href="/">← Back</a></nav>
<main><h1>Financial Disclaimer</h1><div class="meta"><strong>LegacyLift LLC</strong> &nbsp;·&nbsp; Effective: {effective_date}</div>
<div class="big-warning"><h2>⚠ Not Financial Advice</h2><p>LegacyLift is a financial <strong>education</strong> platform. We are not licensed financial advisors, investment advisors, brokers, or any type of financial professional. Nothing on this platform constitutes personalized financial advice, investment recommendations, or professional financial guidance of any kind.</p></div>
<h2>What LegacyLift Is</h2><p>LegacyLift provides financial literacy education, interactive calculators (Freedom Score, debt projections, retirement estimates), and a community platform. All tools are <strong>educational simulations based on data you enter</strong> — not predictions of future outcomes.</p>
<h2>What LegacyLift Is Not</h2><p>LegacyLift does not provide personalized financial advice, investment recommendations, tax advice, legal advice, or guarantees of any financial outcome.</p>
<h2>Always Consult a Professional</h2><p>Before making significant financial decisions, consult a qualified, licensed professional — a Certified Financial Planner (CFP), CPA, or licensed investment advisor. General educational content cannot account for your specific circumstances.</p>
<h2>No Liability</h2><p>LegacyLift LLC expressly disclaims liability for financial decisions made based on Platform content. All examples are illustrative only. Past examples do not guarantee future results.</p>
<h2>Children's Content</h2><p>Our children's curriculum is for educational purposes only. Parents and guardians are responsible for all financial decisions made on behalf of their children.</p>
<h2>Contact</h2><p><a href="mailto:support@legacylift.app">support@legacylift.app</a></p>
</main>
<footer>&copy; {year} LegacyLift LLC &nbsp;·&nbsp; <a href="/privacy">Privacy</a> &nbsp;·&nbsp; <a href="/terms">Terms</a> &nbsp;·&nbsp; <a href="/disclaimer">Disclaimer</a> &nbsp;·&nbsp; Education only — not financial advice</footer>
</body></html>"""

ABOUT_HTML = """<!DOCTYPE html>
<html lang="en"><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Our Story | LegacyLift</title>
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,400;0,600;1,400&family=DM+Sans:wght@400;500;600&display=swap" rel="stylesheet"/>
<style>*{box-sizing:border-box;margin:0;padding:0}:root{--bg:#09090b;--s1:#111114;--s2:#18181c;--bord:rgba(255,255,255,.08);--text:#f0eeea;--muted:#6b6b78;--gold:#c8a84b;--gold2:#e2c97e;--lift:#7c6af5;--lift2:#a89cf8;--em:#22c98a;--sans:'DM Sans',sans-serif;--serif:'Cormorant Garamond',serif}body{font-family:var(--sans);background:var(--bg);color:var(--text);line-height:1.7}.glow{position:fixed;inset:0;pointer-events:none;background:radial-gradient(ellipse 60% 40% at 20% 20%,rgba(124,106,245,.05),transparent 60%),radial-gradient(ellipse 40% 30% at 80% 80%,rgba(200,168,75,.04),transparent 60%)}nav{background:var(--s1);border-bottom:1px solid var(--bord);padding:1rem 2rem;display:flex;align-items:center;justify-content:space-between;position:sticky;top:0;z-index:10}.brand{font-family:var(--serif);font-size:1.2rem;font-weight:600;background:linear-gradient(135deg,var(--lift2),var(--gold));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;text-decoration:none}.back{font-size:.8rem;color:var(--muted);text-decoration:none}.back:hover{color:var(--text)}.hero{padding:5rem 2rem 3rem;text-align:center;position:relative;overflow:hidden}.hero::before{content:'';position:absolute;inset:0;background:radial-gradient(ellipse 80% 50% at 50% 0%,rgba(124,106,245,.07),transparent 65%);pointer-events:none}.hero-tag{display:inline-block;font-size:.62rem;letter-spacing:.16em;text-transform:uppercase;color:var(--lift2);border:1px solid rgba(124,106,245,.3);padding:.28rem .9rem;border-radius:20px;margin-bottom:1.2rem;background:rgba(124,106,245,.07)}.hero h1{font-family:var(--serif);font-size:clamp(2.2rem,5vw,3.8rem);font-weight:600;color:var(--text);line-height:1.1;margin-bottom:1rem;max-width:680px;margin-left:auto;margin-right:auto}.hero h1 em{font-style:italic;color:var(--gold2)}.hero-sub{font-size:1rem;color:var(--muted);max-width:520px;margin:0 auto;line-height:1.75}.content{max-width:720px;margin:0 auto;padding:0 2rem 4rem;position:relative;z-index:1}.pull-quote{font-family:var(--serif);font-size:1.55rem;font-style:italic;color:var(--text);line-height:1.4;margin:2.5rem 0;padding:1.75rem 2rem;border-left:3px solid var(--gold);background:rgba(200,168,75,.06);border-radius:0 12px 12px 0}h2{font-family:var(--serif);font-size:1.5rem;font-weight:600;color:var(--text);margin:2.5rem 0 .85rem}p{font-size:.92rem;color:var(--muted);margin-bottom:1.1rem;line-height:1.85}p strong{color:var(--text);font-weight:600}.stats{display:grid;grid-template-columns:repeat(3,1fr);gap:1rem;margin:2.5rem 0}.stat-box{background:var(--s2);border:1px solid var(--bord);border-radius:12px;padding:1.25rem;text-align:center}.stat-num{font-family:var(--serif);font-size:2.2rem;font-weight:600;background:linear-gradient(135deg,var(--gold2),var(--lift2));-webkit-background-clip:text;-webkit-text-fill-color:transparent;background-clip:text;line-height:1;margin-bottom:.3rem}.stat-lbl{font-size:.7rem;color:var(--muted);letter-spacing:.06em;text-transform:uppercase}.values{display:grid;grid-template-columns:1fr 1fr;gap:1rem;margin:2rem 0}.vcard{background:var(--s1);border:1px solid var(--bord);border-radius:12px;padding:1.25rem}.vic{font-size:1.4rem;margin-bottom:.6rem}.vtitle{font-family:var(--serif);font-size:1rem;font-weight:600;color:var(--text);margin-bottom:.3rem}.vdesc{font-size:.8rem;color:var(--muted);line-height:1.6}.cta-box{background:linear-gradient(135deg,rgba(124,106,245,.1),rgba(200,168,75,.06));border:1px solid rgba(124,106,245,.2);border-radius:16px;padding:2rem;text-align:center;margin-top:3rem}.cta-box h2{margin:0 0 .6rem;font-size:1.6rem}.cta-box p{margin:0 0 1.4rem;max-width:480px;margin-left:auto;margin-right:auto}.cta-btn{display:inline-block;background:linear-gradient(135deg,var(--lift),#5948d4);color:#fff;padding:.75rem 2rem;border-radius:10px;font-weight:600;font-size:.9rem;text-decoration:none;box-shadow:0 4px 14px rgba(124,106,245,.3)}.cta-btn:hover{box-shadow:0 6px 20px rgba(124,106,245,.45)}footer{text-align:center;padding:2rem;border-top:1px solid var(--bord);font-size:.78rem;color:var(--muted)}footer a{color:var(--muted);text-decoration:none}footer a:hover{color:var(--gold)}@media(max-width:600px){.stats{grid-template-columns:1fr}.values{grid-template-columns:1fr}}</style></head>
<body><div class="glow"></div>
<nav><a class="brand" href="/">LegacyLift</a><a class="back" href="/">← Back</a></nav>
<div class="hero">
  <div class="hero-tag">Our Story</div>
  <h1>Built for my family.<br/><em>Built for yours.</em></h1>
  <p class="hero-sub">LegacyLift didn't start as a business idea. It started as a personal mission — to give my family the financial foundation I wish I'd had earlier.</p>
</div>
<div class="content">
  <p>I didn't grow up with anyone teaching me about money. Nobody sat me down and explained how compound interest worked, or what a 401(k) match meant, or why paying the minimum on a credit card was one of the most expensive decisions a person could make. I learned all of that the hard way — through mistakes, through stress, through years of feeling like I was always one unexpected expense away from losing control.</p>
  <p>When I became a parent, I made a decision: <strong>my kids were not going to learn about money the way I did.</strong></p>
  <div class="pull-quote">"I wanted to build something that could teach my children the habits that would protect them for a lifetime — and help my family break a cycle at the same time."</div>
  <p>When I looked for tools to do that, I found apps that tracked spending but didn't teach why it mattered. Calculators that gave numbers with no context. Content written for finance professionals, not for real families figuring it out in real time. So I built LegacyLift.</p>
  <h2>What I Wanted to Create</h2>
  <p>One platform that does three things nobody was doing together: <strong>give you a real-time picture of your financial health</strong>, <strong>give you the tools to fix it</strong>, and <strong>teach your whole family the habits that make wealth stick across generations.</strong></p>
  <p>The Freedom Score was designed to show you exactly where you stand — no guessing, no anxiety, just a clear number and a clear path forward. The debt engine tells you the exact month you'll be free and what moves the needle most. And the education curriculum — the 12 lessons for adults and children — was written with one question in mind: <strong>what would I want my own kids to know?</strong></p>
  <div class="stats">
    <div class="stat-box"><div class="stat-num">12</div><div class="stat-lbl">Curated Lessons</div></div>
    <div class="stat-box"><div class="stat-num">2</div><div class="stat-lbl">Generations Served</div></div>
    <div class="stat-box"><div class="stat-num">1</div><div class="stat-lbl">Mission: Your Legacy</div></div>
  </div>
  <h2>The Name</h2>
  <p>LegacyLift means exactly what it says. A legacy is what you leave behind. A lift is what you give your family when you decide to stop reacting to money and start building something intentional. Every family deserves that lift — regardless of where they're starting from.</p>
  <div class="values">
    <div class="vcard"><div class="vic">🏛</div><div class="vtitle">Wealth is built in habits, not moments</div><div class="vdesc">The difference between struggle and freedom is almost never one big decision. It's dozens of small, consistent ones — made on purpose.</div></div>
    <div class="vcard"><div class="vic">👨‍👩‍👧‍👦</div><div class="vtitle">Education belongs to every generation</div><div class="vdesc">Financial literacy shouldn't start at 35 when the damage is done. It should start at 8, with a jar labeled SAVE, and grow from there.</div></div>
    <div class="vcard"><div class="vic">🔍</div><div class="vtitle">Clarity beats complexity</div><div class="vdesc">The financial industry profits from confusion. We believe simple, honest tools explained in plain language change more lives than complicated ones.</div></div>
    <div class="vcard"><div class="vic">❤</div><div class="vtitle">This is personal</div><div class="vdesc">LegacyLift was built by someone who needed it, for people who need it. That's not a marketing line — it's the whole reason this exists.</div></div>
  </div>
  <div class="cta-box">
    <h2>Your legacy starts with one decision.</h2>
    <p>Join families building real financial habits — for themselves, and for their children.</p>
    <a href="/signup" class="cta-btn">Begin Your Legacy — Free →</a>
  </div>
</div>
<footer>&copy; {year} LegacyLift LLC &nbsp;·&nbsp; <a href="/privacy">Privacy</a> &nbsp;·&nbsp; <a href="/terms">Terms</a> &nbsp;·&nbsp; <a href="/disclaimer">Disclaimer</a> &nbsp;·&nbsp; Education only — not financial advice</footer>
</body></html>"""


@app.get("/privacy")
def privacy(): return _legal_page("Privacy Policy", PRIVACY_HTML)

@app.get("/terms")
def terms(): return _legal_page("Terms of Service", TERMS_HTML)

@app.get("/disclaimer")
def disclaimer(): return _legal_page("Disclaimer", DISCLAIMER_HTML)

@app.get("/about")
def about(): return _legal_page("Our Story", ABOUT_HTML)

@app.get("/support")
def support():
    body = f"""<div style="max-width:520px;margin:3rem auto;text-align:center;">
<div class="section-label">Support</div>
<h1 style="margin-bottom:.75rem;font-family:'Cormorant Garamond',serif;">We're here to help.</h1>
<p class="muted" style="margin-bottom:1.5rem;font-size:.9rem;line-height:1.75;">
  Have a question, bug report, billing issue, or just want to share your progress?<br/>
  We read and respond to every message — usually within 24 hours.
</p>
<div class="card" style="text-align:left;">
  <div class="section-label">Send Us a Message</div>
  <p style="font-size:.88rem;color:var(--muted);margin-bottom:1rem;">
    Email us directly at
    <a href="mailto:support@legacylift.app" style="color:var(--green);font-family:var(--mono);">
      support@legacylift.app
    </a>
  </p>
  <div style="background:var(--surface2);border-radius:8px;padding:1rem;">
    <div style="font-size:.72rem;letter-spacing:.08em;text-transform:uppercase;color:var(--muted);margin-bottom:.5rem;">Include in your email</div>
    <ul style="list-style:none;font-size:.82rem;color:var(--text);line-height:2.1;">
      <li>✦ Your account email address</li>
      <li>✦ Description of the issue or question</li>
      <li>✦ Screenshot if it's a display issue</li>
    </ul>
  </div>
</div>
<p class="muted" style="margin-top:1.5rem;font-size:.78rem;">
  For billing issues and refund requests, please include your order number or the email used to subscribe.
</p>
</div>"""
    return page("Support", body)


# ── Cancellation Save Flow ────────────────────────────────────────────────────

@app.get("/subscription/cancel")
def cancel_page():
    gate = require_login()
    if gate: return gate
    u = me()
    body = f"""<div style="max-width:520px;margin:3rem auto;">
<div class="section-label">Before You Go</div>
<h1 style="margin-bottom:.75rem;">We hate to see you leave.</h1>
<div class="card" style="border-color:var(--amber);margin-bottom:1rem;">
  <div class="section-label" style="color:var(--amber);">What you'll lose access to</div>
  <ul style="list-style:none;font-size:.84rem;line-height:2.2;color:var(--text);">
    <li>— Debt Destruction Engine + exact payoff schedule</li>
    <li>— CSV Cashflow Analyzer + leak detection</li>
    <li>— Retirement Legacy Planner + milestone projections</li>
    <li>— 7 Premium Education Lessons (Adults + Children's Curriculum)</li>
    <li>— Your full transaction history and Freedom Score history</li>
  </ul>
</div>
<div class="card card-gold" style="margin-bottom:1rem;">
  <div class="section-label">Stay one more month — on us</div>
  <p style="font-size:.85rem;color:var(--text);margin-bottom:1rem;line-height:1.7;">
    We'll credit your next billing cycle at 100%. No charge.
    Give LegacyLift 30 more days to prove its value to your family.
  </p>
  <form method="post" action="{url_for('apply_discount')}">
    <button class="btn btn-primary btn-block" type="submit">Apply My Free Month →</button>
  </form>
</div>
<div style="text-align:center;">
  <form method="post" action="{url_for('confirm_cancel')}">
    <button class="btn btn-danger" type="submit" style="font-size:.74rem;">
      No thanks — cancel my subscription
    </button>
  </form>
</div>
</div>"""
    return page("Cancel Subscription", body)

@app.post("/subscription/discount")
def apply_discount():
    gate = require_login()
    if gate: return gate
    u = me()
    if STRIPE_ENABLED and u.stripe_subscription_id:
        try:
            coupon = stripe.Coupon.create(
                percent_off=100, duration="once", name="LegacyLift Retention"
            )
            sub = stripe.Subscription.retrieve(u.stripe_subscription_id)
            stripe.SubscriptionItem.modify(
                sub["items"]["data"][0]["id"],
                discounts=[{"coupon": coupon.id}]
            )
        except Exception:
            pass
    flash("Free month applied! Your next billing cycle is on us. Thank you for staying.", "success")
    return redirect(url_for("dashboard"))

@app.post("/subscription/confirm-cancel")
def confirm_cancel():
    gate = require_login()
    if gate: return gate
    u = me()
    if STRIPE_ENABLED and u.stripe_subscription_id:
        try:
            stripe.Subscription.modify(u.stripe_subscription_id, cancel_at_period_end=True)
        except Exception:
            pass
    u.subscription_status = "canceling"
    db.session.commit()
    flash("Subscription cancelled. You retain access until the end of your current billing period.", "info")
    return redirect(url_for("dashboard"))


# ── Updated BASE template with LegacyLift branding + legal footer links ──────
# (The BASE template already exists above — this updates the footer)
# Replace the footer line in BASE with:
# <footer>LEGACYLIFT &nbsp;|&nbsp; EDUCATION ONLY — NOT FINANCIAL ADVICE
# &nbsp;|&nbsp; <a href="/privacy" style="color:var(--muted)">Privacy</a>
# &nbsp;·&nbsp; <a href="/terms" style="color:var(--muted)">Terms</a>
# &nbsp;·&nbsp; <a href="/disclaimer" style="color:var(--muted)">Disclaimer</a>
# &nbsp;·&nbsp; <a href="/about" style="color:var(--muted)">Our Story</a>
# &nbsp;·&nbsp; <a href="/support" style="color:var(--muted)">Support</a>
# &nbsp;|&nbsp; &copy; {{ year }} LegacyLift LLC</footer>


# ── Boot ──────────────────────────────────────────────────────────────────────

with app.app_context():
    db.create_all()
    seed_edu_lessons()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT","5000")), debug=True)


# ═══════════════════════════════════════════════════════════════════════════════
# FEATURE ADDITIONS: Password Reset + Onboarding Wizard + Weekly Report Email
# ═══════════════════════════════════════════════════════════════════════════════

DAILY_TIPS = [
    "Track every purchase under $10 for one week. Most people find $300+ in invisible spending.",
    "The 24-hour rule: wait a day before any unplanned purchase over $30.",
    "Your employer 401(k) match is the only guaranteed return in investing. Never leave it unclaimed.",
    "An emergency fund doesn't earn money — it saves you from losing money to high-interest debt.",
    "Compound interest works against you on debt and for you on savings. Both are always running.",
    "Pay yourself first means the transfer happens before you see the money. Automation wins.",
    "The DOLP method: divide each debt balance by its minimum payment. Attack lowest number first.",
    "A $6,000 credit card at 22% APR costs $1,320/year in interest at the minimum payment.",
    "Increasing your savings rate 1% every 3 months is barely noticeable. Over 3 years, it's transformative.",
    "Teaching a child about needs vs wants before age 10 changes their relationship with money forever.",
    "The best investment account is the one you actually contribute to consistently.",
    "Freedom Score below 50? Focus only on cashflow and emergency fund. Everything else waits.",
    "Subscriptions are the silent wealth killer. Audit yours every 6 months.",
    "The Three Jar System works for adults too. Give every dollar a job before you spend it.",
    "Index funds outperform 90% of actively managed funds over 20 years. Low cost wins.",
    "Your debt-free date is a real date. Calculate it. Write it down. Tell someone.",
    "The most powerful word in personal finance: automatic. Automate everything good.",
    "A budget isn't a restriction. It's permission to spend on what matters without guilt.",
    "Rent vs buy: homeowners build 40x more wealth over 30 years than renters.",
    "Starting investing at 25 vs 35 produces roughly double the retirement portfolio.",
    "Generosity trains your brain for abundance. Regular givers earn more over their lifetimes.",
    "Variable spending is your biggest lever. Fixed bills are hard to cut. Variable is flexible.",
    "The net worth number is more motivating than the income number. Start tracking it monthly.",
    "One financial win shared in community multiplies motivation for everyone who reads it.",
    "Your Freedom Score is a compass, not a grade. Use it to navigate, not to judge.",
]


def _daily_tip_card(tip: str) -> str:
    return f"""<div style="background:rgba(124,106,245,.07);border:1px solid rgba(124,106,245,.18);
  border-radius:10px;padding:.85rem 1.1rem;margin-bottom:1.1rem;
  display:flex;align-items:flex-start;gap:.75rem;">
  <span style="font-size:1rem;flex-shrink:0;margin-top:.05rem;">&#x1F4A1;</span>
  <div>
    <div style="font-size:.6rem;letter-spacing:.1em;text-transform:uppercase;
      color:#a89cf8;margin-bottom:.25rem;">Today's Insight</div>
    <div style="font-size:.82rem;color:var(--text);line-height:1.65;font-style:italic;">
      "{safe(tip)}"
    </div>
  </div>
</div>"""


def needs_onboarding(u) -> bool:
    ob = OnboardingState.query.filter_by(user_id=u.id).first()
    return ob is None or not ob.completed


def get_onboarding_state(u) -> OnboardingState:
    ob = OnboardingState.query.filter_by(user_id=u.id).first()
    if not ob:
        ob = OnboardingState(user_id=u.id)
        db.session.add(ob); db.session.commit()
    return ob


ONBOARDING_STEPS = [
    {"num": 1, "title": "Your monthly income",    "tag": "Step 1 of 5 — Income"},
    {"num": 2, "title": "Your monthly bills",     "tag": "Step 2 of 5 — Fixed Costs"},
    {"num": 3, "title": "Debt & emergency fund",  "tag": "Step 3 of 5 — Debt & Savings"},
    {"num": 4, "title": "Your primary goal",      "tag": "Step 4 of 5 — Goals"},
    {"num": 5, "title": "Your family",            "tag": "Step 5 of 5 — Household"},
]


def _ob_bar(current: int) -> str:
    dots = "".join(
        f'<div style="height:5px;border-radius:3px;transition:all .3s;'
        f'background:{"var(--green)" if i <= current else "rgba(255,255,255,.08)"};'
        f'flex:{"3" if i == current else "1"};"></div>'
        for i in range(1, 6)
    )
    return f'<div style="display:flex;gap:4px;align-items:center;margin-bottom:1.5rem;">{dots}</div>'


# ── PASSWORD RESET ────────────────────────────────────────────────────────────

def _send_reset_email(to_email: str, reset_url: str) -> bool:
    print(f"[PASSWORD RESET] To: {to_email} | URL: {reset_url}")
    try:
        from flask_mail import Mail, Message as MailMsg
        mail = Mail(app)
        msg = MailMsg(
            subject="Reset your LegacyLift password",
            recipients=[to_email],
            html=f"""<!DOCTYPE html><html><head><style>
body{{font-family:Arial,sans-serif;background:#09090b;color:#f0eeea;margin:0;padding:0}}
.wrap{{max-width:520px;margin:2rem auto;padding:1.5rem}}
.brand{{font-size:1.4rem;font-weight:700;color:#e2c97e;margin-bottom:1.25rem}}
.card{{background:#111114;border:1px solid rgba(255,255,255,.08);border-radius:12px;padding:1.75rem}}
h1{{font-size:1.4rem;color:#f0eeea;margin:0 0 .65rem}}
p{{color:#6b6b78;font-size:.88rem;line-height:1.75;margin:0 0 .85rem}}
.btn{{display:inline-block;background:#7c6af5;color:#fff;padding:.8rem 1.75rem;
      border-radius:10px;text-decoration:none;font-weight:700;font-size:.88rem;margin:.75rem 0}}
.small{{font-size:.75rem;color:#3a3a42;margin-top:1rem}}
.footer{{text-align:center;margin-top:1.25rem;font-size:.72rem;color:#3a3a42}}
</style></head><body><div class="wrap">
<div class="brand">LegacyLift</div>
<div class="card">
<h1>Reset your password</h1>
<p>We received a request to reset the password for your account ({to_email}).</p>
<p>Click the button below. This link expires in <strong style="color:#f0eeea">1 hour</strong>.</p>
<a href="{reset_url}" class="btn">Reset My Password &rarr;</a>
<p class="small">If you didn't request this, you can safely ignore this email. Your password won't change.</p>
</div>
<div class="footer">LegacyLift LLC &middot; Education only &mdash; not financial advice</div>
</div></body></html>"""
        )
        mail.send(msg)
        return True
    except Exception as e:
        print(f"[EMAIL ERROR] {e}")
        return False


@app.get("/forgot-password")
def forgot_password():
    body = f"""<div style="max-width:420px;margin:3rem auto;">
<div class="section-label">Account Recovery</div>
<h1 style="margin-bottom:.4rem;">Forgot your password?</h1>
<p class="muted" style="font-size:.86rem;margin-bottom:1.5rem;line-height:1.7;">
  Enter your account email and we'll send you a secure reset link.
</p>
<div class="card">
  <form method="post" action="{url_for('forgot_password_post')}">
    <div class="form-group">
      <label>Email Address</label>
      <input type="email" name="email" required autofocus placeholder="your@email.com">
    </div>
    <button class="btn btn-primary btn-block" type="submit">Send Reset Link &rarr;</button>
  </form>
  <hr>
  <p class="muted" style="font-size:.78rem;text-align:center;">
    Remember it? <a href="{url_for('login')}">Sign in</a>
  </p>
</div>
</div>"""
    return page("Forgot Password", body)


@app.post("/forgot-password")
def forgot_password_post():
    email = (request.form.get("email") or "").strip().lower()
    body = f"""<div style="max-width:420px;margin:3rem auto;text-align:center;">
<div style="font-size:2.5rem;margin-bottom:1rem;">&#x1F4EC;</div>
<div class="section-label">Check Your Email</div>
<h1 style="margin-bottom:.75rem;">Reset link sent.</h1>
<p class="muted" style="font-size:.88rem;line-height:1.75;margin-bottom:1.25rem;">
  If an account exists for <strong style="color:var(--text)">{safe(email)}</strong>,
  you'll receive a password reset link shortly. Check your spam folder if it doesn't arrive.
</p>
<p class="muted" style="font-size:.78rem;margin-bottom:1.5rem;">The link expires in 1 hour.</p>
<a class="btn" href="{url_for('login')}">Back to Login</a>
</div>"""
    u = User.query.filter_by(email=email).first()
    if u:
        raw_token  = secrets.token_urlsafe(32)
        token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
        expires_at = dt.datetime.utcnow() + dt.timedelta(hours=1)
        PasswordResetToken.query.filter_by(user_id=u.id, used=False).update({"used": True})
        db.session.add(PasswordResetToken(user_id=u.id, token_hash=token_hash, expires_at=expires_at))
        db.session.commit()
        reset_url = f"{APP_URL}{url_for('reset_password', token=raw_token)}"
        _send_reset_email(u.email, reset_url)
    return page("Reset Link Sent", body)


@app.get("/reset-password/<token>")
def reset_password(token: str):
    token_hash = hashlib.sha256(token.encode()).hexdigest()
    prt = PasswordResetToken.query.filter_by(token_hash=token_hash, used=False).first()
    if not prt or prt.expires_at < dt.datetime.utcnow():
        body = f"""<div style="max-width:420px;margin:3rem auto;text-align:center;">
<div style="font-size:2.5rem;margin-bottom:1rem;">&#x26A0;&#xFE0F;</div>
<h1 style="margin-bottom:.75rem;">Link expired.</h1>
<p class="muted" style="margin-bottom:1.5rem;font-size:.88rem;line-height:1.7;">
  This reset link has expired or already been used. Reset links are valid for 1 hour.
</p>
<a class="btn btn-primary" href="{url_for('forgot_password')}">Request New Link &rarr;</a>
</div>"""
        return page("Invalid Reset Link", body)
    body = f"""<div style="max-width:420px;margin:3rem auto;">
<div class="section-label">Account Recovery</div>
<h1 style="margin-bottom:.4rem;">Choose a new password</h1>
<p class="muted" style="font-size:.86rem;margin-bottom:1.5rem;">At least 8 characters.</p>
<div class="card">
  <form method="post" action="{url_for('reset_password_post', token=token)}">
    <div class="form-group">
      <label>New Password</label>
      <input type="password" name="password" minlength="8" required autofocus placeholder="At least 8 characters">
    </div>
    <div class="form-group">
      <label>Confirm Password</label>
      <input type="password" name="confirm" minlength="8" required placeholder="Repeat your password">
    </div>
    <button class="btn btn-primary btn-block" type="submit">Set New Password &rarr;</button>
  </form>
</div>
</div>"""
    return page("Set New Password", body)


@app.post("/reset-password/<token>")
def reset_password_post(token: str):
    token_hash = hashlib.sha256(token.encode()).hexdigest()
    prt = PasswordResetToken.query.filter_by(token_hash=token_hash, used=False).first()
    if not prt or prt.expires_at < dt.datetime.utcnow():
        flash("Reset link expired. Please request a new one.", "warning")
        return redirect(url_for("forgot_password"))
    password = request.form.get("password") or ""
    confirm  = request.form.get("confirm") or ""
    if len(password) < 8:
        flash("Password must be at least 8 characters.", "warning")
        return redirect(url_for("reset_password", token=token))
    if password != confirm:
        flash("Passwords don't match. Please try again.", "warning")
        return redirect(url_for("reset_password", token=token))
    u = db.session.get(User, prt.user_id)
    if not u:
        flash("Account not found.", "warning")
        return redirect(url_for("login"))
    u.set_password(password)
    prt.used = True
    db.session.commit()
    session["uid"] = u.id
    flash("Password updated. You're now logged in.", "success")
    return redirect(url_for("dashboard"))


# ── ONBOARDING WIZARD ─────────────────────────────────────────────────────────

@app.get("/onboarding")
def onboarding():
    gate = require_login()
    if gate: return gate
    u = me()
    ob = get_onboarding_state(u)
    if ob.completed:
        return redirect(url_for("dashboard"))
    return redirect(url_for("onboarding_step", step=ob.step or 1))


@app.get("/onboarding/<int:step>")
def onboarding_step(step: int):
    gate = require_login()
    if gate: return gate
    u = me()
    ob = get_onboarding_state(u)
    if ob.completed:
        return redirect(url_for("dashboard"))
    step = max(1, min(5, step))
    p    = ensure_profile(u)
    bar  = _ob_bar(step)
    meta = ONBOARDING_STEPS[step - 1]

    if step == 1:
        form_html = f"""
<p class="muted" style="font-size:.86rem;margin-bottom:1.25rem;line-height:1.7;">
  We use this to calculate your Freedom Score and personalize your dashboard.
  No one else sees this data.
</p>
<form method="post" action="{url_for('onboarding_save', step=1)}">
  <div class="form-group">
    <label>Monthly Take-Home Income (after tax)</label>
    <input type="number" name="monthly_income" min="0" step="100"
           value="{int(p.monthly_income) if p.monthly_income else ''}"
           placeholder="e.g. 4500" required autofocus>
    <p class="muted" style="font-size:.74rem;margin-top:.3rem;">Include all sources — salary, freelance, side income.</p>
  </div>
  <button class="btn btn-primary btn-block" type="submit">Continue &rarr;</button>
</form>"""

    elif step == 2:
        form_html = f"""
<p class="muted" style="font-size:.86rem;margin-bottom:1.25rem;line-height:1.7;">
  Bills that come every month regardless — rent, utilities, insurance, subscriptions.
</p>
<form method="post" action="{url_for('onboarding_save', step=2)}">
  <div class="grid grid-2" style="gap:.75rem;margin-bottom:1rem;">
    <div class="form-group">
      <label>Housing (rent or mortgage)</label>
      <input type="number" name="housing" min="0" step="50" placeholder="e.g. 1400">
    </div>
    <div class="form-group">
      <label>Utilities &amp; Phone</label>
      <input type="number" name="utilities" min="0" step="10" placeholder="e.g. 180">
    </div>
    <div class="form-group">
      <label>Insurance (all types)</label>
      <input type="number" name="insurance" min="0" step="10" placeholder="e.g. 250">
    </div>
    <div class="form-group">
      <label>Subscriptions &amp; Streaming</label>
      <input type="number" name="subscriptions" min="0" step="5" placeholder="e.g. 85">
    </div>
  </div>
  <div style="display:flex;gap:.75rem;">
    <a class="btn" href="{url_for('onboarding_step', step=1)}">Back</a>
    <button class="btn btn-primary" type="submit" style="flex:1">Continue &rarr;</button>
  </div>
</form>"""

    elif step == 3:
        form_html = f"""
<p class="muted" style="font-size:.86rem;margin-bottom:1.25rem;line-height:1.7;">
  Total of all minimum monthly debt payments — credit cards, car, student loans.
  Don't include housing (captured in step 2).
</p>
<form method="post" action="{url_for('onboarding_save', step=3)}">
  <div class="form-group">
    <label>Total Monthly Debt Minimums</label>
    <input type="number" name="debt_minimums" min="0" step="10"
           value="{int(p.debt_minimums) if p.debt_minimums else ''}"
           placeholder="e.g. 485" autofocus>
    <p class="muted" style="font-size:.74rem;margin-top:.3rem;">Estimate is fine. You can add individual debts in the Debt Engine later.</p>
  </div>
  <div class="form-group">
    <label>Current Emergency Fund Balance ($)</label>
    <input type="number" name="emergency_fund" min="0" step="100"
           value="{int(p.emergency_fund_current) if p.emergency_fund_current else ''}"
           placeholder="e.g. 800">
  </div>
  <div style="display:flex;gap:.75rem;">
    <a class="btn" href="{url_for('onboarding_step', step=2)}">Back</a>
    <button class="btn btn-primary" type="submit" style="flex:1">Continue &rarr;</button>
  </div>
</form>"""

    elif step == 4:
        goals = [
            ("eliminate_debt",   "&#x1F4A3;", "Eliminate my debt",          "Pay off credit cards, loans — become completely debt-free"),
            ("emergency_fund",   "&#x1F6E1;", "Build my emergency fund",    "Get 3&ndash;6 months of expenses saved as a financial buffer"),
            ("start_investing",  "&#x1F4C8;", "Start or grow my investing", "Begin building long-term wealth through consistent investing"),
            ("family_education", "&#x1F4DA;", "Educate my family",          "Complete the curriculum with my household and teach my kids"),
            ("improve_cashflow", "&#x1F4B9;", "Fix my monthly cashflow",    "Reduce spending leaks and increase my monthly surplus"),
        ]
        opts = "".join(f"""<label style="display:flex;align-items:flex-start;gap:.75rem;
  padding:.8rem 1rem;border:1px solid var(--border);border-radius:8px;
  cursor:pointer;margin-bottom:.4rem;transition:all .15s;"
  onmouseover="this.style.borderColor='var(--green)';this.style.background='rgba(0,232,122,.04)'"
  onmouseout="this.style.borderColor='var(--border)';this.style.background='transparent'">
  <input type="radio" name="primary_goal" value="{val}"
         style="width:auto;margin-top:.25rem;accent-color:var(--green);" required>
  <div>
    <div style="font-weight:600;font-size:.88rem;color:var(--text);">{ic} {label}</div>
    <div style="font-size:.76rem;color:var(--muted);margin-top:.12rem;">{desc}</div>
  </div>
</label>""" for val, ic, label, desc in goals)
        form_html = f"""
<p class="muted" style="font-size:.86rem;margin-bottom:1.25rem;">
  We'll customize your dashboard, lesson recommendations, and alerts around this goal.
</p>
<form method="post" action="{url_for('onboarding_save', step=4)}">
  {opts}
  <div style="display:flex;gap:.75rem;margin-top:.85rem;">
    <a class="btn" href="{url_for('onboarding_step', step=3)}">Back</a>
    <button class="btn btn-primary" type="submit" style="flex:1">Continue &rarr;</button>
  </div>
</form>"""

    elif step == 5:
        form_html = f"""
<p class="muted" style="font-size:.86rem;margin-bottom:1.25rem;line-height:1.7;">
  LegacyLift has a full curriculum for children aged 5&ndash;14.
  This helps us personalize your family's experience.
</p>
<form method="post" action="{url_for('onboarding_save', step=5)}">
  <div class="form-group">
    <label>What should we call your household?</label>
    <input type="text" name="display_name" maxlength="80" autofocus
           value="{safe(p.display_name) if p.display_name != 'Household' else ''}"
           placeholder="e.g. The Williams Family">
  </div>
  <div class="form-group">
    <label>Do you have children to include?</label>
    <select name="has_kids">
      <option value="yes">Yes &mdash; I want to use the children's curriculum</option>
      <option value="no">Not right now</option>
    </select>
  </div>
  <div class="form-group">
    <label>How do you get paid?</label>
    <select name="pay_frequency">
      <option value="biweekly">Every 2 weeks (biweekly)</option>
      <option value="weekly">Weekly</option>
      <option value="semimonthly">Twice a month (semi-monthly)</option>
      <option value="monthly">Monthly</option>
    </select>
  </div>
  <div style="display:flex;gap:.75rem;">
    <a class="btn" href="{url_for('onboarding_step', step=4)}">Back</a>
    <button class="btn btn-primary" type="submit" style="flex:1">Build My Dashboard &rarr;</button>
  </div>
</form>"""

    body = f"""
<div style="max-width:520px;margin:2.5rem auto;">
  <div style="margin-bottom:1.1rem;">
    <div style="font-size:.6rem;letter-spacing:.14em;text-transform:uppercase;
      color:var(--green);margin-bottom:.18rem;">{meta['tag']}</div>
    <h1 style="font-size:1.5rem;line-height:1.1;">{meta['title']}</h1>
  </div>
  {bar}
  <div class="card">{form_html}</div>
  <p class="muted" style="font-size:.7rem;text-align:center;margin-top:1rem;">
    All data is private, encrypted, and never shared. &nbsp;
    <a href="{url_for('dashboard')}" style="color:var(--muted);">Skip setup &rarr;</a>
  </p>
</div>"""
    return page(f"Setup {step}/5", body)


@app.post("/onboarding/<int:step>")
def onboarding_save(step: int):
    gate = require_login()
    if gate: return gate
    u  = me()
    p  = ensure_profile(u)
    ob = get_onboarding_state(u)

    def fv(name, default=0.0):
        try: return max(0.0, float(request.form.get(name) or default))
        except: return default

    if step == 1:
        p.monthly_income = fv("monthly_income")

    elif step == 2:
        p.fixed_bills = fv("housing") + fv("utilities") + fv("insurance") + fv("subscriptions")

    elif step == 3:
        p.debt_minimums          = fv("debt_minimums")
        p.emergency_fund_current = fv("emergency_fund")
        if p.monthly_income > 0 and p.emergency_fund_target <= 500.0:
            p.emergency_fund_target = p.monthly_income * 3

    elif step == 4:
        goal = (request.form.get("primary_goal") or "eliminate_debt").strip()
        defaults = {"eliminate_debt": 50, "emergency_fund": 50, "start_investing": 300,
                    "family_education": 100, "improve_cashflow": 100}
        if p.monthly_investing_target <= 100:
            p.monthly_investing_target = float(defaults.get(goal, 100))

    elif step == 5:
        name = (request.form.get("display_name") or "").strip()[:80]
        if name: p.display_name = name
        score_val, _ = compute_freedom_score(u)
        db.session.add(ScoreHistory(user_id=u.id, score=score_val))
        ob.completed = True

    ob.step       = min(step + 1, 5)
    ob.updated_at = dt.datetime.utcnow()
    db.session.commit()

    if step == 5:
        flash(f"Welcome to LegacyLift! Your Freedom Score is ready.", "success")
        return redirect(url_for("dashboard"))
    return redirect(url_for("onboarding_step", step=step + 1))


# ── WEEKLY SCORE REPORT ───────────────────────────────────────────────────────

def _get_lesson_rec(parts: dict) -> dict:
    candidates = [
        ("net_points",  25, "A2", "Pay Yourself First",       "Your cashflow needs attention — automate your savings first."),
        ("ef_points",   20, "A6", "Protection Before Optimization", "Building your safety net is the most urgent priority."),
        ("debt_points", 20, "A3", "The DOLP Debt System",      "Your debt burden is holding your score down."),
        ("inv_points",  15, "A4", "The 401(k) Match",          "You may be leaving free employer money on the table."),
        ("leak_points", 20, "A1", "The Latte Factor",          "Spending leaks are your biggest opportunity right now."),
    ]
    weakest = min(candidates, key=lambda x: (parts.get(x[0], x[1]) / x[1]))
    return {"key": weakest[2], "title": weakest[3], "reason": weakest[4]}


def _generate_weekly_email_html(u, score: int, prev_score: int, parts: dict, alerts: list) -> str:
    change       = score - prev_score
    change_str   = f"+{change}" if change > 0 else str(change)
    change_color = "#22c98a" if change > 0 else "#f05555" if change < 0 else "#c8a84b"
    change_word  = "up" if change > 0 else "down" if change < 0 else "unchanged"
    score_color  = "#22c98a" if score >= 60 else "#c8a84b" if score >= 35 else "#f05555"
    p            = ensure_profile(u)
    rec          = _get_lesson_rec(parts)
    tip_idx      = dt.date.today().timetuple().tm_yday % len(DAILY_TIPS)
    tip          = DAILY_TIPS[tip_idx]
    top_alert    = alerts[0] if alerts else "Your financial position is looking strong this week."
    done_edu     = EduProgress.query.filter_by(user_id=u.id, completed=True).count()
    all_edu      = EduLesson.query.count()
    net_pct      = int(parts.get("net_points", 0) / 25 * 100)
    ef_pct       = int(parts.get("ef_points",  0) / 20 * 100)

    return f"""<!DOCTYPE html>
<html><head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
<style>
*{{box-sizing:border-box;margin:0;padding:0}}
body{{font-family:Arial,sans-serif;background:#09090b;color:#f0eeea;line-height:1.6}}
.wrap{{max-width:560px;margin:0 auto;padding:1.25rem 1rem}}
.hdr{{text-align:center;padding:1.75rem 0 1.25rem;border-bottom:1px solid rgba(255,255,255,.08)}}
.brand{{font-size:1.5rem;font-weight:700;color:#e2c97e;margin-bottom:.2rem}}
.week-lbl{{font-size:.68rem;letter-spacing:.12em;text-transform:uppercase;color:#6b6b78}}
.score-hero{{background:#111114;border:1px solid rgba(255,255,255,.08);border-radius:14px;
             padding:1.75rem;margin:1.25rem 0;text-align:center}}
.score-num{{font-size:4rem;font-weight:700;line-height:1;color:{score_color}}}
.score-lbl{{font-size:.72rem;color:#6b6b78;letter-spacing:.1em;text-transform:uppercase;margin-top:.2rem;margin-bottom:.85rem}}
.chg{{display:inline-block;background:{change_color}22;color:{change_color};
      border:1px solid {change_color}44;border-radius:20px;padding:.25rem .85rem;font-size:.8rem;font-weight:700}}
.stats{{display:flex;gap:.5rem;margin-top:.85rem}}
.stat{{flex:1;background:#18181c;border-radius:8px;padding:.6rem;text-align:center}}
.sv{{font-size:1.1rem;font-weight:700;color:#f0eeea}}
.sl{{font-size:.6rem;color:#6b6b78;letter-spacing:.06em;text-transform:uppercase;margin-top:.12rem}}
.sec-lbl{{font-size:.62rem;letter-spacing:.12em;text-transform:uppercase;color:#6b6b78;margin-bottom:.65rem;margin-top:1.1rem}}
.card{{background:#111114;border:1px solid rgba(255,255,255,.08);border-radius:10px;padding:1rem 1.1rem;margin-bottom:.5rem}}
.alert-txt{{font-size:.84rem;color:#f0eeea;line-height:1.65}}
.tip{{background:rgba(124,106,245,.08);border:1px solid rgba(124,106,245,.2);border-radius:10px;padding:1rem 1.1rem}}
.tip-txt{{font-size:.82rem;color:#d8d0ff;line-height:1.65;font-style:italic}}
.rec{{background:rgba(200,168,75,.08);border:1px solid rgba(200,168,75,.2);border-radius:10px;padding:1rem 1.1rem}}
.rec-r{{font-size:.76rem;color:#c8a84b;margin-bottom:.4rem}}
.rec-t{{font-size:.9rem;font-weight:700;color:#f0eeea;margin-bottom:.75rem}}
.btn{{display:inline-block;padding:.7rem 1.6rem;border-radius:9px;text-decoration:none;font-weight:700;font-size:.85rem}}
.btn-purple{{background:#7c6af5;color:#fff}}
.btn-gold{{background:#c8a84b;color:#0a0a0c}}
.cta-row{{text-align:center;margin:1.25rem 0;display:flex;gap:.6rem;justify-content:center;flex-wrap:wrap}}
.footer{{text-align:center;padding:1.25rem 0;border-top:1px solid rgba(255,255,255,.06);margin-top:1.25rem}}
.footer a{{color:#6b6b78;font-size:.72rem;text-decoration:none}}
</style></head>
<body><div class="wrap">
<div class="hdr">
  <div class="brand">LegacyLift</div>
  <div class="week-lbl">Weekly Legacy Report &middot; {dt.date.today().strftime("%B %d, %Y")}</div>
</div>
<div class="score-hero">
  <div class="score-num">{score}</div>
  <div class="score-lbl">Freedom Score / 100</div>
  <div class="chg">{change_str} points {change_word} this week</div>
  <div class="stats">
    <div class="stat"><div class="sv">{done_edu}/{all_edu}</div><div class="sl">Lessons</div></div>
    <div class="stat"><div class="sv">{net_pct}%</div><div class="sl">Cashflow</div></div>
    <div class="stat"><div class="sv">{ef_pct}%</div><div class="sl">Emergency Fund</div></div>
  </div>
</div>
<div class="sec-lbl">// Top alert this week</div>
<div class="card"><div class="alert-txt">{safe(top_alert)}</div></div>
<div class="sec-lbl">// Recommended next lesson</div>
<div class="rec">
  <div class="rec-r">{safe(rec['reason'])}</div>
  <div class="rec-t">&#x1F4DA; {safe(rec['title'])}</div>
  <a href="{APP_URL}/learn/education" class="btn btn-purple">Start This Lesson &rarr;</a>
</div>
<div class="sec-lbl">// This week's insight</div>
<div class="tip"><div class="tip-txt">"{safe(tip)}"</div></div>
<div class="cta-row">
  <a href="{APP_URL}/dashboard" class="btn btn-gold">View Full Dashboard &rarr;</a>
</div>
<div class="footer">
  <a href="{APP_URL}/dashboard">Dashboard</a> &nbsp;&middot;&nbsp;
  <a href="{APP_URL}/learn/education">Education</a> &nbsp;&middot;&nbsp;
  <a href="mailto:support@legacylift.app">Support</a> &nbsp;&middot;&nbsp;
  <a href="{APP_URL}/account/email-preferences">Unsubscribe</a>
  <p style="font-size:.7rem;color:#2a2a32;margin-top:.65rem">
    LegacyLift LLC &middot; Education only &mdash; not financial advice
  </p>
</div>
</div></body></html>"""


def send_weekly_reports():
    """
    Send weekly score reports to all users.
    Trigger via Render cron job (every Monday 9am UTC):
      startCommand: python -c "from legacylift_app import app,send_weekly_reports; app.app_context().push(); send_weekly_reports()"
    Or manually via /admin/send-weekly-reports
    """
    users = User.query.all()
    sent  = 0
    for u in users:
        try:
            score, parts = compute_freedom_score(u)
            alerts       = generate_alerts(u)
            prev = ScoreHistory.query.filter_by(user_id=u.id)\
                       .order_by(ScoreHistory.recorded_at.desc()).offset(1).first()
            prev_score = prev.score if prev else score
            db.session.add(ScoreHistory(user_id=u.id, score=score))
            html = _generate_weekly_email_html(u, score, prev_score, parts, alerts)
            # Swap this line for your real email provider (SendGrid, Mailgun, etc):
            print(f"[WEEKLY REPORT] {u.email} — Score: {score} ({'+' if score-prev_score>=0 else ''}{score-prev_score})")
            sent += 1
        except Exception as e:
            print(f"[WEEKLY REPORT ERROR] user {u.id}: {e}")
    db.session.commit()
    return sent


@app.get("/admin/send-weekly-reports")
def admin_send_weekly_reports():
    u = me()
    if not u or u.org_role != "admin": abort(403)
    count = send_weekly_reports()
    flash(f"Weekly reports sent for {count} users.", "success")
    return redirect(url_for("dashboard"))


@app.get("/account/email-preferences")
def email_preferences():
    gate = require_login()
    if gate: return gate
    body = f"""<div style="max-width:480px;margin:3rem auto;">
<div class="section-label">Account Settings</div>
<h1 style="margin-bottom:.5rem;">Email Preferences</h1>
<p class="muted" style="font-size:.85rem;margin-bottom:1.5rem;">Control which emails you receive from LegacyLift.</p>
<div class="card">
  <form method="post" action="{url_for('email_preferences_save')}">
    {"".join(_ep_toggle(n,t,d,c) for n,t,d,c in [
      ("weekly_report","Weekly Freedom Score Report","Every Monday: your score, change, top alert, and lesson recommendation.",True),
      ("lesson_reminder","Lesson Reminders","A gentle nudge when you haven't completed a lesson in 2 weeks.",True),
      ("community_digest","Community Wins Digest","Weekly highlights from the LegacyLift community.",False),
      ("product_updates","Product Updates","New lessons, features, and improvements.",True),
    ])}
    <div style="border-top:1px solid var(--border);padding-top:1rem;margin-top:.25rem;">
      <button class="btn btn-primary btn-block" type="submit">Save Preferences &rarr;</button>
    </div>
  </form>
</div>
<div style="text-align:center;margin-top:1rem;">
  <a href="{url_for('dashboard')}" class="muted" style="font-size:.8rem;">&#x2190; Back to Dashboard</a>
</div>
</div>"""
    return page("Email Preferences", body)


def _ep_toggle(name, title, desc, default):
    return f"""<div style="display:flex;align-items:flex-start;justify-content:space-between;
  gap:1rem;padding:.85rem 0;border-bottom:1px solid var(--border);">
  <div>
    <div style="font-size:.86rem;font-weight:600;color:var(--text);margin-bottom:.12rem;">{title}</div>
    <div style="font-size:.76rem;color:var(--muted);">{desc}</div>
  </div>
  <input type="checkbox" name="{name}" {"checked" if default else ""}
         style="width:18px;height:18px;accent-color:var(--green);cursor:pointer;margin-top:.2rem;flex-shrink:0;">
</div>"""


@app.post("/account/email-preferences")
def email_preferences_save():
    gate = require_login()
    if gate: return gate
    flash("Email preferences saved.", "success")
    return redirect(url_for("email_preferences"))

