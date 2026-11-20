# PLANNING.md — ToolShare

## Part 1: Business Case
* **Problem:** Residents on a street own tools (drills, ladders, lawn equipment) they only use occasionally, while neighbours buy duplicates they rarely need.
* **Who has this problem:** A resident on a residential street who needs a tool for a weekend project and does not want to buy one for one-time use.
* **Why now:** Informal group chats bury listings quickly; a dedicated shared list keeps tools visible and requests organised.
* **Success looks like:** A neighbour can list a tool and another neighbour can find and request it in under two minutes.

## Part 2: Use Case
* **Primary actors:** Tool Owner and Tool Borrower (same user account model performing owner or borrower actions).
* **User stories:**
  1. As a Tool Owner, I want to list a tool with a photo and description, so that neighbours know it is available.
  2. As a Borrower, I want to search tools by name and filter by category, so that I can find what I need quickly.
  3. As a Borrower, I want to request a tool, so that the Owner can approve or decline it.
  4. As an Owner, I want to approve or decline requests and mark borrowed tools as returned, so that I control who borrows my tools and keep availability accurate.
  5. As a user, I want to register and log in securely, so that my listings, requests, and contact email are tied to my account.
* **Happy-path flow:**
  1. User registers or logs in.
  2. Owner creates a tool listing with photo and description.
  3. Borrower searches/filters and opens the tool detail page.
  4. Borrower submits a borrow request.
  5. Owner approves the request.
  6. Tool is marked `Borrowed` until the Owner marks it as returned (`Available`).

## Part 3: Tech Spec
* **Stack:** Django + Pillow (SQLite for development/deployment, or PostgreSQL where configured) — Django's built-in auth and admin speed up account and data management.
* **Data model:**
  * `User`: Built-in Django user model (`username`, `email`, `password`).
  * `Tool`: `owner` (FK to `AUTH_USER_MODEL`), `name`, `category` (`power`, `hand`, `garden`, `ladders`, `cleaning`, `other`), `description`, `photo` (`ImageField` -> `tools/`), `status` (`available` / `borrowed`).
  * `BorrowRequest`: `tool` (FK to `Tool`), `borrower` (FK to `AUTH_USER_MODEL`), `status` (`pending` / `approved` / `declined`), `requested_at` (`auto_now_add=True`).
* **Core screens / endpoints:** `/`, `/tools/<id>/`, `/tools/add/`, `/tools/<id>/request/` (POST), `/requests/mine/`, `/tools/mine/`, `/requests/<id>/approve/` (POST), `/requests/<id>/decline/` (POST), `/tools/<id>/returned/` (POST), `/register/`, `/accounts/login/`, `/accounts/logout/` (POST), `/admin/`.
* **Out of scope:** No in-app messaging (use contact email), no payments/deposits, no ratings/reviews, no native mobile app (web only).
* **Stretch goal:** Email notification to the borrower when a request is approved.
```[cite: 3, 15, 16, 17, 19, 20, 28, 34, 39, 44]

---

## 2. Exact Code & Deployment Steps — ToolShare (`lizelgr.pythonanywhere.com`)

### Key Code Files (`tools/`)

* **`tools/models.py`**
```python
from django.conf import settings
from django.db import models


class Tool(models.Model):
    CATEGORY_CHOICES = [
        ('power', 'Power tools'),
        ('hand', 'Hand tools'),
        ('garden', 'Garden & lawn'),
        ('ladders', 'Ladders'),
        ('cleaning', 'Cleaning'),
        ('other', 'Other'),
    ]
    STATUS_CHOICES = [
        ('available', 'Available'),
        ('borrowed', 'Borrowed'),
    ]

    owner = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='tools'
    )
    name = models.CharField(max_length=100)
    category = models.CharField(max_length=20, choices=CATEGORY_CHOICES)
    description = models.TextField(blank=True)
    photo = models.ImageField(upload_to='tools/', blank=True)
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='available')

    def __str__(self):
        return self.name


class BorrowRequest(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('approved', 'Approved'),
        ('declined', 'Declined'),
    ]

    tool = models.ForeignKey(Tool, on_delete=models.CASCADE, related_name='requests')
    borrower = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name='borrow_requests'
    )
    status = models.CharField(max_length=10, choices=STATUS_CHOICES, default='pending')
    requested_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f'{self.borrower} -> {self.tool} ({self.status})'
```[cite: 19, 20]

* **`tools/forms.py`**
```python
from django import forms
from django.contrib.auth import get_user_model
from django.contrib.auth.forms import UserCreationForm
from .models import Tool


class RegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)

    class Meta(UserCreationForm.Meta):
        model = get_user_model()
        fields = ('username', 'email')


class ToolForm(forms.ModelForm):
    class Meta:
        model = Tool
        fields = ['name', 'category', 'description', 'photo']
```[cite: 25, 28]

* **`tools/views.py`**
```python
from django.contrib import messages
from django.contrib.auth import login
from django.contrib.auth.decorators import login_required
from django.core.mail import send_mail
from django.db import transaction
from django.shortcuts import get_object_or_404, redirect, render
from django.views.decorators.http import require_POST
from .forms import RegisterForm, ToolForm
from .models import BorrowRequest, Tool


def tool_list(request):
    tools = Tool.objects.select_related('owner').order_by('name')
    q = request.GET.get('q', '').strip()
    category = request.GET.get('category', '')
    if q:
        tools = tools.filter(name__icontains=q)
    if category:
        tools = tools.filter(category=category)
    return render(
        request,
        'tools/tool_list.html',
        {
            'tools': tools,
            'categories': Tool.CATEGORY_CHOICES,
            'q': q,
            'selected_category': category,
        },
    )


def tool_detail(request, pk):
    tool = get_object_or_404(Tool, pk=pk)
    can_request = False
    already_requested = False
    if request.user.is_authenticated and tool.owner != request.user:
        already_requested = tool.requests.filter(
            borrower=request.user, status='pending'
        ).exists()
        can_request = tool.status == 'available' and not already_requested
    return render(
        request,
        'tools/tool_detail.html',
        {
            'tool': tool,
            'can_request': can_request,
            'already_requested': already_requested,
        },
    )


def register(request):
    if request.method == 'POST':
        form = RegisterForm(request.POST)
        if form.is_valid():
            user = form.save()
            login(request, user)
            return redirect('tool_list')
    else:
        form = RegisterForm()
    return render(request, 'registration/register.html', {'form': form})


@login_required
def add_tool(request):
    if request.method == 'POST':
        form = ToolForm(request.POST, request.FILES)
        if form.is_valid():
            tool = form.save(commit=False)
            tool.owner = request.user
            tool.save()
            messages.success(request, 'Your tool is now listed.')
            return redirect('tool_detail', pk=tool.pk)
    else:
        form = ToolForm()
    return render(request, 'tools/tool_form.html', {'form': form})


@login_required
@require_POST
def request_tool(request, pk):
    tool = get_object_or_404(Tool, pk=pk)
    if tool.owner == request.user:
        messages.error(request, "That's your own tool.")
    elif tool.status != 'available':
        messages.error(request, 'That tool is currently borrowed.')
    elif tool.requests.filter(borrower=request.user, status='pending').exists():
        messages.info(request, 'You already have a pending request for this tool.')
    else:
        BorrowRequest.objects.create(tool=tool, borrower=request.user)
        messages.success(request, 'Request sent. The owner can now approve or decline it.')
        return redirect('my_requests')
    return redirect('tool_detail', pk=tool.pk)


@login_required
def my_requests(request):
    borrow_requests = (
        BorrowRequest.objects.filter(borrower=request.user)
        .select_related('tool')
        .order_by('-requested_at')
    )
    return render(request, 'tools/my_requests.html', {'borrow_requests': borrow_requests})


@login_required
def my_tools(request):
    tools = Tool.objects.filter(owner=request.user).order_by('name')
    incoming = (
        BorrowRequest.objects.filter(tool__owner=request.user)
        .select_related('tool', 'borrower')
        .order_by('-requested_at')
    )
    return render(request, 'tools/my_tools.html', {'tools': tools, 'incoming': incoming})


@login_required
@require_POST
def decide_request(request, pk, decision):
    borrow_request = get_object_or_404(BorrowRequest, pk=pk, tool__owner=request.user)
    if borrow_request.status != 'pending':
        messages.info(request, 'That request has already been handled.')
    elif decision == 'approve':
        if borrow_request.tool.status != 'available':
            messages.error(request, 'That tool is already out on loan.')
        else:
            with transaction.atomic():
                borrow_request.status = 'approved'
                borrow_request.save()
                borrow_request.tool.status = 'borrowed'
                borrow_request.tool.save()
            messages.success(request, 'Request approved.')
            if borrow_request.borrower.email:
                send_mail(
                    subject=f'Your request for {borrow_request.tool.name} was approved',
                    message=(
                        f'Hi {borrow_request.borrower.username},\n\n'
                        f'{request.user.username} approved your request to borrow '
                        f'"{borrow_request.tool.name}".\n'
                        f'Contact them at {request.user.email} to arrange pick-up.'
                    ),
                    from_email=None,
                    recipient_list=[borrow_request.borrower.email],
                )
    else:
        borrow_request.status = 'declined'
        borrow_request.save()
        messages.success(request, 'Request declined.')
    return redirect('my_tools')


@login_required
@require_POST
def mark_returned(request, pk):
    tool = get_object_or_404(Tool, pk=pk, owner=request.user)
    tool.status = 'available'
    tool.save()
    messages.success(request, f'{tool.name} is available again.')
    return redirect('my_tools')
```[cite: 21, 22, 26, 28, 30, 31, 34, 39]

* **`tools/urls.py`**
```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.tool_list, name='tool_list'),
    path('tools/<int:pk>/', views.tool_detail, name='tool_detail'),
    path('register/', views.register, name='register'),
    path('tools/add/', views.add_tool, name='add_tool'),
    path('tools/<int:pk>/request/', views.request_tool, name='request_tool'),
    path('requests/mine/', views.my_requests, name='my_requests'),
    path('tools/mine/', views.my_tools, name='my_tools'),
    path('tools/<int:pk>/returned/', views.mark_returned, name='mark_returned'),
    path(
        'requests/<int:pk>/approve/',
        views.decide_request,
        {'decision': 'approve'},
        name='approve_request',
    ),
    path(
        'requests/<int:pk>/decline/',
        views.decide_request,
        {'decision': 'decline'},
        name='decline_request',
    ),
]
```[cite: 22, 26, 29, 32, 34, 35]

### Deploying / Updating ToolShare on PythonAnywhere (`lizelgr.pythonanywhere.com`)

1. **Push from local machine** (with `venv/` and `media/` in `.gitignore` and `PLANNING.md` + tests committed):
```bash
git add .
git commit -m "ToolShare: all user stories working, with tests"
git push -u origin main
```[cite: 19, 41]
2. **In PythonAnywhere Bash console** (under account `lizelg`):
```bash
git clone https://github.com/Lizelober/tool-share-app.git
cd tool-share-app
mkvirtualenv toolshare-venv --python=/usr/bin/python3.13
pip install -r requirements.txt
python manage.py migrate
python manage.py createsuperuser
python manage.py collectstatic
```[cite: 8]
3. **In PythonAnywhere Web tab**:
   * Create a **Manual configuration** web app on Python 3.13 for `lizelgr.pythonanywhere.com`[cite: 8, 9].
   * Set **Source code** and **Working directory** to `/home/lizelg/tool-share-app`, and **Virtualenv** to `toolshare-venv`[cite: 8, 9].
   * In the WSGI file (`/var/www/lizelgr_pythonanywhere_com_wsgi.py`), set `DJANGO_SETTINGS_MODULE = 'config.settings'`, `SECRET_KEY`, `DEBUG = 'False'`, and `ALLOWED_HOSTS = 'lizelgr.pythonanywhere.com'`[cite: 9, 11].
   * Map `/static/` to `/home/lizelg/tool-share-app/staticfiles` (and `/media/` to `/home/lizelg/tool-share-app/media` if serving uploaded photos), then click **Reload**[cite: 9, 29].

---

## 3. Exact Deployment & Update Steps — Learning Log (`lizelg.pythonanywhere.com`)

1. **Prepare `settings.py` & Git** (in `django_project_two`):
   * Ensure `SECRET_KEY`, `DEBUG`, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`, and `STATIC_ROOT = BASE_DIR / 'staticfiles'` read from `os.environ` at the end of `settings.py`[cite: 5].
   * Keep `requirements.txt` updated (`pip freeze > requirements.txt`) and exclude virtualenvs, `__pycache__/`, `*.sqlite3`, `staticfiles/`, and `.DS_Store` in `.gitignore`[cite: 6].
2. **Initial PythonAnywhere setup** (under `lizelg` for `[https://lizelg.pythonanywhere.com/](https://lizelg.pythonanywhere.com/)`):
   * Clone `[https://github.com/Lizelober/django_project_two.git](https://github.com/Lizelober/django_project_two.git)`, create `learning-log-venv` with Python 3.13, and install `requirements.txt`[cite: 7, 8].
   * Run `python manage.py migrate`, `python manage.py createsuperuser`, and `python manage.py collectstatic`[cite: 8].
   * Configure a **Manual configuration** web app on Python 3.13, set `ALLOWED_HOSTS = 'lizelg.pythonanywhere.com'` in the WSGI file, map `/static/` to `/home/lizelg/<project_dir>/staticfiles`, and click **Reload**[cite: 8, 9, 11].
3. **Ongoing update routine**:
```bash
cd ~/django_project_two
workon learning-log-venv
git pull
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --no-input
```[cite: 10]
   Then click **Reload** on the **Web** tab[cite: 10].

Would you like to generate the HTML templates (`base.html`, `tool_list.html`, `tool_detail.html`, `tool_form.html`, `my_requests.html`, `my_tools.html`) or `tools/tests.py` next?
