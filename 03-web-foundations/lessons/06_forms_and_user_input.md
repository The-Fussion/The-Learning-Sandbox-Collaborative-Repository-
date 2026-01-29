# Forms and User Input with Python

## Table of Contents
1. [Introduction to Forms](#introduction-to-forms)
2. [HTML Form Basics](#html-form-basics)
3. [Processing Form Data](#processing-form-data)
4. [Form Validation](#form-validation)
5. [Flask-WTF Forms](#flask-wtf-forms)
6. [Django Forms](#django-forms)
7. [CSRF Protection](#csrf-protection)
8. [File Uploads](#file-uploads)
9. [Security Best Practices](#security-best-practices)
10. [Advanced Form Techniques](#advanced-form-techniques)

---

## Introduction to Forms

**Web forms** are the primary way users interact with web applications, allowing them to submit data, create accounts, post content, and more.

### Why Forms Matter

1. **User Interaction**: Enable two-way communication
2. **Data Collection**: Gather information from users
3. **Authentication**: Login and registration
4. **Content Creation**: Posts, comments, uploads
5. **E-commerce**: Orders, payments, shipping

### Form Processing Flow

```
1. User requests form (GET)
   ↓
2. Server renders form HTML
   ↓
3. User fills out form
   ↓
4. User submits form (POST)
   ↓
5. Server receives data
   ↓
6. Server validates data
   ↓
7. If valid: Process and save
   If invalid: Show errors and form again
```

---

## HTML Form Basics

### Basic Form Structure

```html
<form method="POST" action="/submit">
    <!-- Text input -->
    <label for="username">Username:</label>
    <input type="text" id="username" name="username" required>
    
    <!-- Email input -->
    <label for="email">Email:</label>
    <input type="email" id="email" name="email" required>
    
    <!-- Password input -->
    <label for="password">Password:</label>
    <input type="password" id="password" name="password" required>
    
    <!-- Textarea -->
    <label for="bio">Bio:</label>
    <textarea id="bio" name="bio" rows="4"></textarea>
    
    <!-- Select dropdown -->
    <label for="country">Country:</label>
    <select id="country" name="country">
        <option value="">Select...</option>
        <option value="us">United States</option>
        <option value="uk">United Kingdom</option>
        <option value="ca">Canada</option>
    </select>
    
    <!-- Radio buttons -->
    <fieldset>
        <legend>Gender:</legend>
        <label><input type="radio" name="gender" value="male"> Male</label>
        <label><input type="radio" name="gender" value="female"> Female</label>
        <label><input type="radio" name="gender" value="other"> Other</label>
    </fieldset>
    
    <!-- Checkboxes -->
    <fieldset>
        <legend>Interests:</legend>
        <label><input type="checkbox" name="interests" value="coding"> Coding</label>
        <label><input type="checkbox" name="interests" value="music"> Music</label>
        <label><input type="checkbox" name="interests" value="sports"> Sports</label>
    </fieldset>
    
    <!-- File upload -->
    <label for="avatar">Avatar:</label>
    <input type="file" id="avatar" name="avatar" accept="image/*">
    
    <!-- Hidden field -->
    <input type="hidden" name="user_id" value="123">
    
    <!-- Submit button -->
    <button type="submit">Submit</button>
</form>
```

### HTTP Methods

**GET:**
- Data in URL query string
- Visible in browser history
- Can be bookmarked
- Limited data size
- Use for: searching, filtering, reading

**POST:**
- Data in request body
- Not visible in URL
- Cannot be bookmarked
- No size limit (practically)
- Use for: creating, updating, sensitive data

```html
<!-- GET form (search) -->
<form method="GET" action="/search">
    <input type="text" name="q" placeholder="Search...">
    <button type="submit">Search</button>
</form>
<!-- URL: /search?q=python -->

<!-- POST form (login) -->
<form method="POST" action="/login">
    <input type="text" name="username">
    <input type="password" name="password">
    <button type="submit">Login</button>
</form>
<!-- Data sent in request body, not visible in URL -->
```

---

## Processing Form Data

### Flask Form Processing

```python
from flask import Flask, request, render_template, redirect, url_for, flash

app = Flask(__name__)
app.secret_key = 'your-secret-key-here'

# Simple form handling
@app.route('/contact', methods=['GET', 'POST'])
def contact():
    if request.method == 'POST':
        # Access form data
        name = request.form.get('name')
        email = request.form.get('email')
        message = request.form.get('message')
        
        # Process the data
        print(f"Contact from {name} ({email}): {message}")
        
        flash('Message sent successfully!', 'success')
        return redirect(url_for('contact'))
    
    # GET request - show the form
    return render_template('contact.html')

# Form with validation
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        email = request.form.get('email', '').strip()
        password = request.form.get('password', '')
        confirm_password = request.form.get('confirm_password', '')
        
        errors = []
        
        # Validation
        if not username or len(username) < 3:
            errors.append('Username must be at least 3 characters')
        
        if not email or '@' not in email:
            errors.append('Valid email is required')
        
        if not password or len(password) < 8:
            errors.append('Password must be at least 8 characters')
        
        if password != confirm_password:
            errors.append('Passwords do not match')
        
        if errors:
            for error in errors:
                flash(error, 'error')
            return render_template('register.html')
        
        # If valid, process registration
        # create_user(username, email, password)
        flash('Registration successful!', 'success')
        return redirect(url_for('login'))
    
    return render_template('register.html')

# Handling multiple values (checkboxes)
@app.route('/preferences', methods=['GET', 'POST'])
def preferences():
    if request.method == 'POST':
        # Get list of values
        interests = request.form.getlist('interests')
        
        # Get single value with default
        newsletter = request.form.get('newsletter', 'off')
        
        print(f"Interests: {interests}")
        print(f"Newsletter: {newsletter}")
        
        return redirect(url_for('preferences'))
    
    return render_template('preferences.html')

# File upload
@app.route('/upload', methods=['GET', 'POST'])
def upload():
    if request.method == 'POST':
        # Check if file was uploaded
        if 'file' not in request.files:
            flash('No file uploaded', 'error')
            return redirect(request.url)
        
        file = request.files['file']
        
        # Check if filename is empty
        if file.filename == '':
            flash('No file selected', 'error')
            return redirect(request.url)
        
        # Save file
        if file:
            filename = secure_filename(file.filename)
            file.save(os.path.join('uploads', filename))
            flash('File uploaded successfully', 'success')
            return redirect(url_for('upload'))
    
    return render_template('upload.html')

if __name__ == '__main__':
    app.run(debug=True)
```

### Django Form Processing

```python
# views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from django.views.decorators.http import require_http_methods

@require_http_methods(["GET", "POST"])
def contact(request):
    if request.method == 'POST':
        # Access POST data
        name = request.POST.get('name', '')
        email = request.POST.get('email', '')
        message = request.POST.get('message', '')
        
        # Validation
        errors = []
        if not name:
            errors.append('Name is required')
        if not email:
            errors.append('Email is required')
        if not message:
            errors.append('Message is required')
        
        if errors:
            for error in errors:
                messages.error(request, error)
        else:
            # Process the data
            # send_contact_email(name, email, message)
            messages.success(request, 'Message sent successfully!')
            return redirect('contact')
    
    return render(request, 'contact.html')

# Class-based view
from django.views import View

class RegisterView(View):
    template_name = 'register.html'
    
    def get(self, request):
        return render(request, self.template_name)
    
    def post(self, request):
        username = request.POST.get('username', '').strip()
        email = request.POST.get('email', '').strip()
        password = request.POST.get('password', '')
        
        # Validation and processing
        # ...
        
        return redirect('login')
```

---

## Form Validation

### Manual Validation (Flask)

```python
from flask import Flask, request, render_template, flash
import re

app = Flask(__name__)
app.secret_key = 'secret'

def validate_email(email):
    """Validate email format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def validate_password(password):
    """Validate password strength"""
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    if not re.search(r'[A-Z]', password):
        return False, "Password must contain uppercase letter"
    if not re.search(r'[a-z]', password):
        return False, "Password must contain lowercase letter"
    if not re.search(r'\d', password):
        return False, "Password must contain a digit"
    return True, None

@app.route('/signup', methods=['GET', 'POST'])
def signup():
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        email = request.form.get('email', '').strip()
        password = request.form.get('password', '')
        age = request.form.get('age', '')
        
        errors = []
        
        # Username validation
        if not username:
            errors.append('Username is required')
        elif len(username) < 3:
            errors.append('Username must be at least 3 characters')
        elif not username.isalnum():
            errors.append('Username must be alphanumeric')
        
        # Email validation
        if not email:
            errors.append('Email is required')
        elif not validate_email(email):
            errors.append('Invalid email format')
        
        # Password validation
        if not password:
            errors.append('Password is required')
        else:
            valid, error_msg = validate_password(password)
            if not valid:
                errors.append(error_msg)
        
        # Age validation
        if age:
            try:
                age_int = int(age)
                if age_int < 13:
                    errors.append('You must be at least 13 years old')
                elif age_int > 120:
                    errors.append('Invalid age')
            except ValueError:
                errors.append('Age must be a number')
        
        if errors:
            for error in errors:
                flash(error, 'error')
            return render_template('signup.html', 
                                 username=username, 
                                 email=email)
        
        # Process signup
        flash('Account created successfully!', 'success')
        return redirect(url_for('login'))
    
    return render_template('signup.html')
```

### Server-Side Validation Example

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route('/api/validate', methods=['POST'])
def validate_form():
    """API endpoint for form validation"""
    data = request.get_json()
    errors = {}
    
    # Validate username
    username = data.get('username', '').strip()
    if not username:
        errors['username'] = 'Username is required'
    elif len(username) < 3:
        errors['username'] = 'Username must be at least 3 characters'
    elif User.query.filter_by(username=username).first():
        errors['username'] = 'Username already taken'
    
    # Validate email
    email = data.get('email', '').strip()
    if not email:
        errors['email'] = 'Email is required'
    elif not validate_email(email):
        errors['email'] = 'Invalid email format'
    elif User.query.filter_by(email=email).first():
        errors['email'] = 'Email already registered'
    
    # Validate password
    password = data.get('password', '')
    if not password:
        errors['password'] = 'Password is required'
    else:
        valid, msg = validate_password(password)
        if not valid:
            errors['password'] = msg
    
    if errors:
        return jsonify({'valid': False, 'errors': errors}), 400
    
    return jsonify({'valid': True}), 200
```

---

## Flask-WTF Forms

Flask-WTF integrates WTForms with Flask, providing CSRF protection and validation.

### Installation

```bash
pip install flask-wtf
```

### Basic Flask-WTF Form

```python
from flask import Flask, render_template, redirect, url_for, flash
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, TextAreaField, SelectField, BooleanField, SubmitField
from wtforms.validators import DataRequired, Email, Length, EqualTo, ValidationError

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'

# Define form class
class RegistrationForm(FlaskForm):
    username = StringField('Username', 
                          validators=[
                              DataRequired(),
                              Length(min=3, max=20)
                          ])
    
    email = StringField('Email',
                       validators=[
                           DataRequired(),
                           Email()
                       ])
    
    password = PasswordField('Password',
                            validators=[
                                DataRequired(),
                                Length(min=8)
                            ])
    
    confirm_password = PasswordField('Confirm Password',
                                    validators=[
                                        DataRequired(),
                                        EqualTo('password', message='Passwords must match')
                                    ])
    
    submit = SubmitField('Sign Up')
    
    # Custom validator
    def validate_username(self, username):
        # Check if username exists in database
        user = User.query.filter_by(username=username.data).first()
        if user:
            raise ValidationError('Username already taken. Please choose another.')

class LoginForm(FlaskForm):
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    
    password = PasswordField('Password',
                            validators=[DataRequired()])
    
    remember = BooleanField('Remember Me')
    
    submit = SubmitField('Login')

class PostForm(FlaskForm):
    title = StringField('Title',
                       validators=[DataRequired(), Length(min=5, max=200)])
    
    content = TextAreaField('Content',
                           validators=[DataRequired(), Length(min=10)])
    
    category = SelectField('Category',
                          choices=[
                              ('tech', 'Technology'),
                              ('travel', 'Travel'),
                              ('food', 'Food'),
                              ('other', 'Other')
                          ],
                          validators=[DataRequired()])
    
    submit = SubmitField('Post')

# Routes
@app.route('/register', methods=['GET', 'POST'])
def register():
    form = RegistrationForm()
    
    if form.validate_on_submit():
        # Form is valid, process data
        username = form.username.data
        email = form.email.data
        password = form.password.data
        
        # Create user (hash password first!)
        # user = User(username=username, email=email)
        # user.set_password(password)
        # db.session.add(user)
        # db.session.commit()
        
        flash(f'Account created for {username}!', 'success')
        return redirect(url_for('login'))
    
    return render_template('register.html', form=form)

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    
    if form.validate_on_submit():
        # Validate credentials
        # user = User.query.filter_by(email=form.email.data).first()
        # if user and user.check_password(form.password.data):
        #     login_user(user, remember=form.remember.data)
        #     return redirect(url_for('dashboard'))
        
        flash('Login successful!', 'success')
        return redirect(url_for('dashboard'))
    
    return render_template('login.html', form=form)

@app.route('/post/new', methods=['GET', 'POST'])
def new_post():
    form = PostForm()
    
    if form.validate_on_submit():
        # Create post
        # post = Post(
        #     title=form.title.data,
        #     content=form.content.data,
        #     category=form.category.data,
        #     author=current_user
        # )
        # db.session.add(post)
        # db.session.commit()
        
        flash('Post created!', 'success')
        return redirect(url_for('index'))
    
    return render_template('create_post.html', form=form)

if __name__ == '__main__':
    app.run(debug=True)
```

### Flask-WTF Template

```html
<!-- templates/register.html -->
{% extends "base.html" %}

{% block content %}
<h2>Register</h2>

<form method="POST" action="">
    {{ form.hidden_tag() }}  <!-- CSRF token -->
    
    <div class="form-group">
        {{ form.username.label }}
        {{ form.username(class="form-control") }}
        
        {% if form.username.errors %}
            <div class="errors">
                {% for error in form.username.errors %}
                    <span class="error">{{ error }}</span>
                {% endfor %}
            </div>
        {% endif %}
    </div>
    
    <div class="form-group">
        {{ form.email.label }}
        {{ form.email(class="form-control") }}
        
        {% if form.email.errors %}
            <div class="errors">
                {% for error in form.email.errors %}
                    <span class="error">{{ error }}</span>
                {% endfor %}
            </div>
        {% endif %}
    </div>
    
    <div class="form-group">
        {{ form.password.label }}
        {{ form.password(class="form-control") }}
        
        {% if form.password.errors %}
            <div class="errors">
                {% for error in form.password.errors %}
                    <span class="error">{{ error }}</span>
                {% endfor %}
            </div>
        {% endif %}
    </div>
    
    <div class="form-group">
        {{ form.confirm_password.label }}
        {{ form.confirm_password(class="form-control") }}
        
        {% if form.confirm_password.errors %}
            <div class="errors">
                {% for error in form.confirm_password.errors %}
                    <span class="error">{{ error }}</span>
                {% endfor %}
            </div>
        {% endif %}
    </div>
    
    <div class="form-group">
        {{ form.submit(class="btn btn-primary") }}
    </div>
</form>

<!-- Macro for rendering fields -->
{% macro render_field(field) %}
<div class="form-group">
    {{ field.label }}
    {{ field(class="form-control", **kwargs) }}
    
    {% if field.errors %}
        <div class="errors">
            {% for error in field.errors %}
                <span class="error">{{ error }}</span>
            {% endfor %}
        </div>
    {% endif %}
</div>
{% endmacro %}

<!-- Usage -->
{{ render_field(form.username) }}
{{ render_field(form.email) }}
{% endblock %}
```

### WTForms Validators

```python
from wtforms import StringField, IntegerField, DateField
from wtforms.validators import (
    DataRequired,
    Email,
    Length,
    EqualTo,
    NumberRange,
    Regexp,
    URL,
    Optional,
    InputRequired,
    ValidationError
)

class AdvancedForm(FlaskForm):
    # Required field
    username = StringField('Username',
                          validators=[DataRequired(message='Username is required')])
    
    # Email validation
    email = StringField('Email',
                       validators=[DataRequired(), Email()])
    
    # Length constraints
    bio = TextAreaField('Bio',
                       validators=[Length(min=10, max=500)])
    
    # Password match
    password = PasswordField('Password',
                            validators=[DataRequired(), Length(min=8)])
    confirm = PasswordField('Confirm',
                           validators=[EqualTo('password')])
    
    # Number range
    age = IntegerField('Age',
                      validators=[NumberRange(min=13, max=120)])
    
    # Regex pattern
    phone = StringField('Phone',
                       validators=[
                           Regexp(r'^\d{3}-\d{3}-\d{4}$',
                                 message='Format: 555-555-5555')
                       ])
    
    # URL validation
    website = StringField('Website',
                         validators=[Optional(), URL()])
    
    # Optional vs InputRequired
    nickname = StringField('Nickname',
                          validators=[Optional()])  # Can be empty string
    
    real_name = StringField('Real Name',
                           validators=[InputRequired()])  # Must have data
    
    # Custom validator function
    def validate_age(form, field):
        if field.data and field.data < 18:
            raise ValidationError('Must be 18 or older')
    
    # Another custom validator
    def validate_username(self, username):
        if username.data.lower() in ['admin', 'root', 'system']:
            raise ValidationError('Reserved username')
```

---

## Django Forms

Django has a powerful built-in forms system.

### Django ModelForm

```python
# models.py
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    CATEGORY_CHOICES = [
        ('tech', 'Technology'),
        ('travel', 'Travel'),
        ('food', 'Food'),
        ('other', 'Other'),
    ]
    
    title = models.CharField(max_length=200)
    content = models.TextField()
    category = models.CharField(max_length=20, choices=CATEGORY_CHOICES)
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)
    
    def __str__(self):
        return self.title

# forms.py
from django import forms
from django.contrib.auth.models import User
from django.contrib.auth.forms import UserCreationForm
from .models import Post

class UserRegisterForm(UserCreationForm):
    email = forms.EmailField(required=True)
    
    class Meta:
        model = User
        fields = ['username', 'email', 'password1', 'password2']
    
    def clean_email(self):
        email = self.cleaned_data.get('email')
        if User.objects.filter(email=email).exists():
            raise forms.ValidationError('Email already registered')
        return email

class PostForm(forms.ModelForm):
    class Meta:
        model = Post
        fields = ['title', 'content', 'category']
        widgets = {
            'title': forms.TextInput(attrs={
                'class': 'form-control',
                'placeholder': 'Enter title'
            }),
            'content': forms.Textarea(attrs={
                'class': 'form-control',
                'rows': 10
            }),
            'category': forms.Select(attrs={
                'class': 'form-control'
            })
        }
    
    def clean_title(self):
        title = self.cleaned_data.get('title')
        if len(title) < 5:
            raise forms.ValidationError('Title must be at least 5 characters')
        return title

class ContactForm(forms.Form):
    name = forms.CharField(
        max_length=100,
        widget=forms.TextInput(attrs={
            'class': 'form-control',
            'placeholder': 'Your name'
        })
    )
    
    email = forms.EmailField(
        widget=forms.EmailInput(attrs={
            'class': 'form-control',
            'placeholder': 'your@email.com'
        })
    )
    
    subject = forms.CharField(
        max_length=200,
        widget=forms.TextInput(attrs={
            'class': 'form-control'
        })
    )
    
    message = forms.CharField(
        widget=forms.Textarea(attrs={
            'class': 'form-control',
            'rows': 5
        })
    )
    
    def clean_message(self):
        message = self.cleaned_data.get('message')
        if len(message) < 10:
            raise forms.ValidationError('Message too short')
        return message

# views.py
from django.shortcuts import render, redirect
from django.contrib import messages
from .forms import UserRegisterForm, PostForm, ContactForm

def register(request):
    if request.method == 'POST':
        form = UserRegisterForm(request.POST)
        if form.is_valid():
            form.save()
            username = form.cleaned_data.get('username')
            messages.success(request, f'Account created for {username}!')
            return redirect('login')
    else:
        form = UserRegisterForm()
    
    return render(request, 'register.html', {'form': form})

def create_post(request):
    if request.method == 'POST':
        form = PostForm(request.POST)
        if form.is_valid():
            post = form.save(commit=False)
            post.author = request.user
            post.save()
            messages.success(request, 'Post created successfully!')
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm()
    
    return render(request, 'create_post.html', {'form': form})

def update_post(request, pk):
    post = get_object_or_404(Post, pk=pk)
    
    if request.method == 'POST':
        form = PostForm(request.POST, instance=post)
        if form.is_valid():
            form.save()
            messages.success(request, 'Post updated!')
            return redirect('post_detail', pk=post.pk)
    else:
        form = PostForm(instance=post)
    
    return render(request, 'update_post.html', {'form': form})

def contact(request):
    if request.method == 'POST':
        form = ContactForm(request.POST)
        if form.is_valid():
            # Process the form
            name = form.cleaned_data['name']
            email = form.cleaned_data['email']
            subject = form.cleaned_data['subject']
            message = form.cleaned_data['message']
            
            # Send email or save to database
            messages.success(request, 'Message sent successfully!')
            return redirect('contact')
    else:
        form = ContactForm()
    
    return render(request, 'contact.html', {'form': form})
```

### Django Template Form Rendering

```django
<!-- templates/register.html -->
{% extends "base.html" %}

{% block content %}
<h2>Register</h2>

<form method="POST">
    {% csrf_token %}
    
    <!-- Render entire form -->
    {{ form.as_p }}
    
    <button type="submit">Register</button>
</form>

<!-- OR manual rendering -->
<form method="POST">
    {% csrf_token %}
    
    {% for field in form %}
        <div class="form-group">
            {{ field.label_tag }}
            {{ field }}
            
            {% if field.errors %}
                <div class="errors">
                    {% for error in field.errors %}
                        <span class="error">{{ error }}</span>
                    {% endfor %}
                </div>
            {% endif %}
            
            {% if field.help_text %}
                <small>{{ field.help_text }}</small>
            {% endif %}
        </div>
    {% endfor %}
    
    <button type="submit">Submit</button>
</form>

<!-- OR specific field rendering -->
<form method="POST">
    {% csrf_token %}
    
    <div class="form-group">
        {{ form.username.label_tag }}
        {{ form.username }}
        {{ form.username.errors }}
    </div>
    
    <div class="form-group">
        {{ form.email.label_tag }}
        {{ form.email }}
        {{ form.email.errors }}
    </div>
    
    <button type="submit">Submit</button>
</form>
{% endblock %}
```

---

## CSRF Protection

**CSRF (Cross-Site Request Forgery)** is an attack where malicious sites trick users into performing unwanted actions on authenticated sites.

### How CSRF Works

```
1. User logs into legitimate-site.com
2. User visits malicious-site.com (without logging out)
3. Malicious site contains hidden form:
   <form action="https://legitimate-site.com/transfer" method="POST">
       <input name="amount" value="1000">
       <input name="to" value="attacker">
   </form>
   <script>document.forms[0].submit();</script>
4. Form submits to legitimate site with user's session
5. Money transferred without user's knowledge
```

### Flask CSRF Protection

```python
from flask import Flask
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'

# Enable CSRF protection globally
csrf = CSRFProtect(app)

# All POST requests now require CSRF token
@app.route('/transfer', methods=['POST'])
def transfer():
    # This route is automatically protected
    amount = request.form.get('amount')
    # Process transfer
    return 'Transfer complete'

# Exempt specific routes from CSRF
@csrf.exempt
@app.route('/webhook', methods=['POST'])
def webhook():
    # External webhooks don't have CSRF tokens
    return 'OK'
```

```html
<!-- Flask template with CSRF -->
<form method="POST">
    <input type="hidden" 
           name="csrf_token" 
           value="{{ csrf_token() }}">
    
    <!-- OR with Flask-WTF forms -->
    {{ form.hidden_tag() }}
    
    <!-- Form fields -->
    <button type="submit">Submit</button>
</form>
```

### Django CSRF Protection

Django includes CSRF protection by default.

```python
# settings.py
MIDDLEWARE = [
    'django.middleware.csrf.CsrfViewMiddleware',  # CSRF middleware
    # ... other middleware
]
```

```django
<!-- Django template -->
<form method="POST">
    {% csrf_token %}
    
    <!-- Form fields -->
    <button type="submit">Submit</button>
</form>
```

```python
# Exempting views from CSRF (use carefully!)
from django.views.decorators.csrf import csrf_exempt

@csrf_exempt
def api_endpoint(request):
    # This view doesn't require CSRF token
    return JsonResponse({'status': 'ok'})

# For AJAX requests
from django.views.decorators.csrf import ensure_csrf_cookie

@ensure_csrf_cookie
def get_csrf(request):
    # Returns CSRF token for JavaScript
    return JsonResponse({'csrf_token': request.META.get('CSRF_COOKIE')})
```

### AJAX with CSRF

```javascript
// Flask AJAX with CSRF
function getCSRFToken() {
    return document.querySelector('meta[name="csrf-token"]').content;
}

fetch('/api/data', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': getCSRFToken()
    },
    body: JSON.stringify({data: 'value'})
});

// Django AJAX with CSRF
function getCookie(name) {
    let cookieValue = null;
    if (document.cookie && document.cookie !== '') {
        const cookies = document.cookie.split(';');
        for (let i = 0; i < cookies.length; i++) {
            const cookie = cookies[i].trim();
            if (cookie.substring(0, name.length + 1) === (name + '=')) {
                cookieValue = decodeURIComponent(cookie.substring(name.length + 1));
                break;
            }
        }
    }
    return cookieValue;
}

const csrftoken = getCookie('csrftoken');

fetch('/api/endpoint/', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRFToken': csrftoken
    },
    body: JSON.stringify({data: 'value'})
});
```

---

## File Uploads

### Flask File Upload

```python
from flask import Flask, request, redirect, url_for, flash, render_template
from werkzeug.utils import secure_filename
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = 'secret'
app.config['UPLOAD_FOLDER'] = 'uploads'
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB max file size
app.config['ALLOWED_EXTENSIONS'] = {'txt', 'pdf', 'png', 'jpg', 'jpeg', 'gif'}

def allowed_file(filename):
    """Check if file extension is allowed"""
    return '.' in filename and \
           filename.rsplit('.', 1)[1].lower() in app.config['ALLOWED_EXTENSIONS']

@app.route('/upload', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        # Check if file was uploaded
        if 'file' not in request.files:
            flash('No file part', 'error')
            return redirect(request.url)
        
        file = request.files['file']
        
        # Check if user selected a file
        if file.filename == '':
            flash('No selected file', 'error')
            return redirect(request.url)
        
        # Validate and save file
        if file and allowed_file(file.filename):
            filename = secure_filename(file.filename)
            
            # Add timestamp to avoid overwriting
            import time
            name, ext = os.path.splitext(filename)
            filename = f"{name}_{int(time.time())}{ext}"
            
            filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
            file.save(filepath)
            
            flash(f'File {filename} uploaded successfully!', 'success')
            return redirect(url_for('upload_file'))
        else:
            flash('File type not allowed', 'error')
            return redirect(request.url)
    
    return render_template('upload.html')

# Multiple file upload
@app.route('/upload-multiple', methods=['GET', 'POST'])
def upload_multiple():
    if request.method == 'POST':
        files = request.files.getlist('files')
        
        if not files:
            flash('No files selected', 'error')
            return redirect(request.url)
        
        uploaded = []
        for file in files:
            if file and allowed_file(file.filename):
                filename = secure_filename(file.filename)
                file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
                uploaded.append(filename)
        
        flash(f'{len(uploaded)} files uploaded successfully!', 'success')
        return redirect(url_for('upload_multiple'))
    
    return render_template('upload_multiple.html')

# Image upload with validation
from PIL import Image

@app.route('/upload-image', methods=['GET', 'POST'])
def upload_image():
    if request.method == 'POST':
        if 'image' not in request.files:
            flash('No image uploaded', 'error')
            return redirect(request.url)
        
        file = request.files['image']
        
        if file.filename == '':
            flash('No selected file', 'error')
            return redirect(request.url)
        
        if file and allowed_file(file.filename):
            # Validate it's actually an image
            try:
                img = Image.open(file)
                img.verify()  # Verify it's an image
                
                # Reset file pointer after verify
                file.seek(0)
                
                # Check image dimensions
                width, height = img.size
                if width > 4000 or height > 4000:
                    flash('Image too large (max 4000x4000)', 'error')
                    return redirect(request.url)
                
                # Save original
                filename = secure_filename(file.filename)
                filepath = os.path.join(app.config['UPLOAD_FOLDER'], filename)
                file.save(filepath)
                
                # Create thumbnail
                img = Image.open(filepath)
                img.thumbnail((200, 200))
                thumb_name = f"thumb_{filename}"
                thumb_path = os.path.join(app.config['UPLOAD_FOLDER'], thumb_name)
                img.save(thumb_path)
                
                flash('Image uploaded successfully!', 'success')
                return redirect(url_for('upload_image'))
                
            except Exception as e:
                flash(f'Invalid image file: {str(e)}', 'error')
                return redirect(request.url)
    
    return render_template('upload_image.html')
```

```html
<!-- templates/upload.html -->
<form method="POST" enctype="multipart/form-data">
    {{ form.hidden_tag() }}
    
    <div class="form-group">
        <label for="file">Choose file:</label>
        <input type="file" 
               id="file" 
               name="file" 
               accept=".pdf,.png,.jpg,.jpeg,.gif"
               required>
    </div>
    
    <button type="submit">Upload</button>
</form>

<!-- Multiple files -->
<form method="POST" enctype="multipart/form-data">
    <input type="file" name="files" multiple>
    <button type="submit">Upload Files</button>
</form>

<!-- Image preview before upload -->
<form method="POST" enctype="multipart/form-data">
    <input type="file" 
           id="image" 
           name="image" 
           accept="image/*"
           onchange="previewImage(event)">
    
    <img id="preview" src="" style="display:none; max-width: 300px;">
    
    <button type="submit">Upload Image</button>
</form>

<script>
function previewImage(event) {
    const file = event.target.files[0];
    const preview = document.getElementById('preview');
    
    if (file) {
        const reader = new FileReader();
        reader.onload = function(e) {
            preview.src = e.target.result;
            preview.style.display = 'block';
        }
        reader.readAsDataURL(file);
    }
}
</script>
```

### Django File Upload

```python
# forms.py
from django import forms

class UploadFileForm(forms.Form):
    title = forms.CharField(max_length=50)
    file = forms.FileField()
    
    def clean_file(self):
        file = self.cleaned_data['file']
        
        # Check file size (5MB max)
        if file.size > 5 * 1024 * 1024:
            raise forms.ValidationError('File too large (max 5MB)')
        
        # Check file extension
        allowed_extensions = ['.pdf', '.doc', '.docx', '.txt']
        ext = os.path.splitext(file.name)[1].lower()
        if ext not in allowed_extensions:
            raise forms.ValidationError('File type not allowed')
        
        return file

class ImageUploadForm(forms.Form):
    image = forms.ImageField()
    
    def clean_image(self):
        image = self.cleaned_data['image']
        
        # Validate image
        if image.size > 2 * 1024 * 1024:  # 2MB
            raise forms.ValidationError('Image too large')
        
        # Check dimensions
        from PIL import Image
        img = Image.open(image)
        if img.width > 2000 or img.height > 2000:
            raise forms.ValidationError('Image dimensions too large')
        
        return image

# models.py
from django.db import models

class Document(models.Model):
    title = models.CharField(max_length=200)
    file = models.FileField(upload_to='documents/')
    uploaded_at = models.DateTimeField(auto_now_add=True)

class Photo(models.Model):
    title = models.CharField(max_length=200)
    image = models.ImageField(upload_to='photos/')
    thumbnail = models.ImageField(upload_to='photos/thumbs/', blank=True)
    uploaded_at = models.DateTimeField(auto_now_add=True)

# views.py
from django.shortcuts import render, redirect
from django.core.files.storage import FileSystemStorage
from .forms import UploadFileForm, ImageUploadForm
from .models import Document, Photo

def upload_file(request):
    if request.method == 'POST':
        form = UploadFileForm(request.POST, request.FILES)
        if form.is_valid():
            # Save using FileSystemStorage
            uploaded_file = request.FILES['file']
            fs = FileSystemStorage()
            filename = fs.save(uploaded_file.name, uploaded_file)
            file_url = fs.url(filename)
            
            # OR save to model
            doc = Document(
                title=form.cleaned_data['title'],
                file=uploaded_file
            )
            doc.save()
            
            return redirect('success')
    else:
        form = UploadFileForm()
    
    return render(request, 'upload.html', {'form': form})

def upload_image(request):
    if request.method == 'POST':
        form = ImageUploadForm(request.POST, request.FILES)
        if form.is_valid():
            image = form.cleaned_data['image']
            
            # Create photo object
            photo = Photo(title=request.POST.get('title'), image=image)
            photo.save()
            
            # Create thumbnail
            from PIL import Image
            img = Image.open(photo.image.path)
            img.thumbnail((200, 200))
            
            thumb_path = f"photos/thumbs/thumb_{photo.image.name}"
            img.save(os.path.join(settings.MEDIA_ROOT, thumb_path))
            photo.thumbnail = thumb_path
            photo.save()
            
            return redirect('gallery')
    else:
        form = ImageUploadForm()
    
    return render(request, 'upload_image.html', {'form': form})

# settings.py
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

---

## Security Best Practices

### 1. Input Validation

```python
import re
from flask import request, abort

@app.route('/user/<username>')
def user_profile(username):
    # Validate username format
    if not re.match(r'^[a-zA-Z0-9_]{3,20}, username):
        abort(400, 'Invalid username format')
    
    # Check against database
    user = User.query.filter_by(username=username).first_or_404()
    return render_template('profile.html', user=user)

# Whitelist validation
ALLOWED_CATEGORIES = ['tech', 'travel', 'food', 'lifestyle']

@app.route('/category/<category>')
def category_posts(category):
    if category not in ALLOWED_CATEGORIES:
        abort(404)
    
    posts = Post.query.filter_by(category=category).all()
    return render_template('category.html', posts=posts)
```

### 2. SQL Injection Prevention

```python
# ❌ NEVER do this - SQL injection vulnerability!
username = request.form.get('username')
query = f"SELECT * FROM users WHERE username = '{username}'"
# Attacker could input: ' OR '1'='1

# ✅ ALWAYS use parameterized queries
from flask_sqlalchemy import SQLAlchemy

# SQLAlchemy ORM (safe)
user = User.query.filter_by(username=username).first()

# Raw SQL with parameters (safe)
db.session.execute(
    "SELECT * FROM users WHERE username = :username",
    {"username": username}
)

# Django (always safe with ORM)
User.objects.filter(username=username)
```

### 3. XSS Prevention

```python
# Flask automatically escapes in templates
# {{ user_input }} is safe

# But if you use |safe, validate first!
from markupsafe import Markup
from bleach import clean

@app.route('/post/<int:id>')
def show_post(id):
    post = Post.query.get_or_404(id)
    
    # Allow only safe HTML tags
    allowed_tags = ['p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li']
    clean_content = clean(
        post.content, 
        tags=allowed_tags,
        attributes={'a': ['href', 'title']},
        strip=True
    )
    
    return render_template('post.html', 
                         post=post,
                         content=Markup(clean_content))
```

```html
<!-- Templates auto-escape by default -->
<p>{{ user_comment }}</p>  <!-- Safe: escaped -->

<!-- Only use |safe for trusted content -->
<div>{{ admin_content|safe }}</div>  <!-- Use carefully! -->

<!-- Django also auto-escapes -->
<p>{{ user_input }}</p>  <!-- Safe: escaped -->
```

### 4. Password Security

```python
from werkzeug.security import generate_password_hash, check_password_hash

# Flask
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True)
    password_hash = db.Column(db.String(128))
    
    def set_password(self, password):
        self.password_hash = generate_password_hash(password)
    
    def check_password(self, password):
        return check_password_hash(self.password_hash, password)

# Django (built-in)
from django.contrib.auth.models import User
from django.contrib.auth.hashers import make_password, check_password

# Create user with hashed password
user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='securepassword123'
)

# Check password
from django.contrib.auth import authenticate
user = authenticate(username='john', password='securepassword123')
```

### 5. Rate Limiting

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

app = Flask(__name__)
limiter = Limiter(
    app=app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

# Limit login attempts
@app.route('/login', methods=['POST'])
@limiter.limit("5 per minute")
def login():
    # Login logic
    pass

# Stricter limit for sensitive operations
@app.route('/api/create', methods=['POST'])
@limiter.limit("10 per hour")
def create_resource():
    # Creation logic
    pass

# Django rate limiting
from django.views.decorators.cache import cache_page
from django.core.cache import cache

def rate_limit(key, limit, period):
    def decorator(func):
        def wrapper(request, *args, **kwargs):
            # Get client IP
            ip = request.META.get('REMOTE_ADDR')
            cache_key = f"{key}:{ip}"
            
            # Check rate limit
            requests = cache.get(cache_key, 0)
            if requests >= limit:
                return HttpResponse('Rate limit exceeded', status=429)
            
            # Increment counter
            cache.set(cache_key, requests + 1, period)
            
            return func(request, *args, **kwargs)
        return wrapper
    return decorator

@rate_limit('login', limit=5, period=300)  # 5 per 5 minutes
def login_view(request):
    # Login logic
    pass
```

### 6. Session Security

```python
# Flask session configuration
app.config.update(
    SESSION_COOKIE_SECURE=True,      # HTTPS only
    SESSION_COOKIE_HTTPONLY=True,    # No JavaScript access
    SESSION_COOKIE_SAMESITE='Lax',   # CSRF protection
    PERMANENT_SESSION_LIFETIME=1800  # 30 minutes
)

# Django session settings
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Lax'
SESSION_COOKIE_AGE = 1800  # 30 minutes
```

---

## Advanced Form Techniques

### Dynamic Forms

```python
# Flask-WTF dynamic fields
from flask_wtf import FlaskForm
from wtforms import StringField, FieldList, FormField, SubmitField
from wtforms.validators import DataRequired

class PhoneForm(FlaskForm):
    """Subform for phone numbers"""
    phone = StringField('Phone', validators=[DataRequired()])

class ContactForm(FlaskForm):
    name = StringField('Name', validators=[DataRequired()])
    phones = FieldList(FormField(PhoneForm), min_entries=1)
    submit = SubmitField('Save')

@app.route('/contact', methods=['GET', 'POST'])
def contact():
    form = ContactForm()
    
    if form.validate_on_submit():
        name = form.name.data
        phones = [phone.phone.data for phone in form.phones]
        # Save contact with multiple phones
        return redirect(url_for('index'))
    
    return render_template('contact.html', form=form)
```

```html
<!-- Dynamic form template -->
<form method="POST">
    {{ form.hidden_tag() }}
    
    {{ form.name.label }}
    {{ form.name }}
    
    <div id="phone-numbers">
        {% for phone in form.phones %}
            <div class="phone-entry">
                {{ phone.phone.label }}
                {{ phone.phone }}
            </div>
        {% endfor %}
    </div>
    
    <button type="button" onclick="addPhone()">Add Phone</button>
    {{ form.submit }}
</form>

<script>
function addPhone() {
    // Add new phone field dynamically
    const container = document.getElementById('phone-numbers');
    const count = container.children.length;
    
    const div = document.createElement('div');
    div.className = 'phone-entry';
    div.innerHTML = `
        <label>Phone:</label>
        <input type="text" name="phones-${count}-phone">
    `;
    container.appendChild(div);
}
</script>
```

### Form Wizards (Multi-Step Forms)

```python
from flask import Flask, session, render_template, redirect, url_for

app = Flask(__name__)
app.secret_key = 'secret'

@app.route('/wizard/step1', methods=['GET', 'POST'])
def wizard_step1():
    if request.method == 'POST':
        session['step1'] = {
            'name': request.form.get('name'),
            'email': request.form.get('email')
        }
        return redirect(url_for('wizard_step2'))
    
    return render_template('wizard/step1.html')

@app.route('/wizard/step2', methods=['GET', 'POST'])
def wizard_step2():
    if 'step1' not in session:
        return redirect(url_for('wizard_step1'))
    
    if request.method == 'POST':
        session['step2'] = {
            'address': request.form.get('address'),
            'city': request.form.get('city')
        }
        return redirect(url_for('wizard_step3'))
    
    return render_template('wizard/step2.html')

@app.route('/wizard/step3', methods=['GET', 'POST'])
def wizard_step3():
    if 'step1' not in session or 'step2' not in session:
        return redirect(url_for('wizard_step1'))
    
    if request.method == 'POST':
        # Combine all data
        data = {
            **session['step1'],
            **session['step2'],
            'preferences': request.form.getlist('preferences')
        }
        
        # Save to database
        # create_user(data)
        
        # Clear session
        session.pop('step1', None)
        session.pop('step2', None)
        
        return redirect(url_for('success'))
    
    return render_template('wizard/step3.html',
                         step1=session['step1'],
                         step2=session['step2'])
```

### AJAX Form Submission

```html
<form id="ajax-form">
    <input type="text" name="username" id="username">
    <input type="email" name="email" id="email">
    <button type="submit">Submit</button>
    
    <div id="errors"></div>
    <div id="success"></div>
</form>

<script>
document.getElementById('ajax-form').addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const formData = new FormData(e.target);
    const data = Object.fromEntries(formData);
    
    try {
        const response = await fetch('/api/register', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': getCSRFToken()
            },
            body: JSON.stringify(data)
        });
        
        const result = await response.json();
        
        if (response.ok) {
            document.getElementById('success').textContent = 'Registration successful!';
            document.getElementById('errors').textContent = '';
            e.target.reset();
        } else {
            // Display errors
            const errorDiv = document.getElementById('errors');
            errorDiv.innerHTML = '';
            for (const [field, errors] of Object.entries(result.errors)) {
                errors.forEach(error => {
                    const p = document.createElement('p');
                    p.textContent = `${field}: ${error}`;
                    p.className = 'error';
                    errorDiv.appendChild(p);
                });
            }
        }
    } catch (error) {
        console.error('Error:', error);
    }
});
</script>
```

### Form with Real-Time Validation

```html
<form id="register-form">
    <div class="form-group">
        <label for="username">Username:</label>
        <input type="text" 
               id="username" 
               name="username"
               onblur="validateField('username')">
        <span id="username-error" class="error"></span>
        <span id="username-success" class="success"></span>
    </div>
    
    <div class="form-group">
        <label for="email">Email:</label>
        <input type="email" 
               id="email" 
               name="email"
               onblur="validateField('email')">
        <span id="email-error" class="error"></span>
        <span id="email-success" class="success"></span>
    </div>
    
    <button type="submit">Register</button>
</form>

<script>
async function validateField(fieldName) {
    const input = document.getElementById(fieldName);
    const errorSpan = document.getElementById(`${fieldName}-error`);
    const successSpan = document.getElementById(`${fieldName}-success`);
    
    try {
        const response = await fetch('/api/validate', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRFToken': getCSRFToken()
            },
            body: JSON.stringify({
                field: fieldName,
                value: input.value
            })
        });
        
        const result = await response.json();
        
        if (result.valid) {
            errorSpan.textContent = '';
            successSpan.textContent = '✓';
            input.classList.remove('invalid');
            input.classList.add('valid');
        } else {
            errorSpan.textContent = result.error;
            successSpan.textContent = '';
            input.classList.remove('valid');
            input.classList.add('invalid');
        }
    } catch (error) {
        console.error('Validation error:', error);
    }
}
</script>
```

---

## Summary

### Key Takeaways

**1. Form Basics:**
- Use appropriate HTTP methods (GET for retrieval, POST for submission)
- Always validate on the server side
- Provide clear error messages
- Make forms accessible

**2. Processing Forms:**
- **Flask**: Use `request.form` for POST, `request.args` for GET
- **Django**: Use `request.POST` and `request.GET`
- Use form libraries (Flask-WTF, Django Forms) for complex forms

**3. Validation:**
- Client-side validation for UX
- Server-side validation for security (mandatory)
- Use built-in validators when possible
- Write custom validators for specific needs

**4. Security:**
- **CSRF Protection**: Always enable (automatic in Django, Flask-WTF)
- **XSS Prevention**: Auto-escape output, sanitize HTML input
- **SQL Injection**: Use ORMs or parameterized queries
- **Password Security**: Hash passwords (bcrypt, argon2)
- **Rate Limiting**: Prevent abuse and brute force
- **File Uploads**: Validate type, size, and content

**5. Best Practices:**
- Keep forms simple and focused
- Group related fields
- Use appropriate input types
- Provide helpful error messages
- Show field-level and form-level errors
- Pre-fill forms when editing
- Confirm destructive actions
- Save partial progress for long forms

**6. Advanced Techniques:**
- Dynamic forms for flexible inputs
- Multi-step wizards for complex workflows
- AJAX submissions for better UX
- Real-time validation for immediate feedback
- File uploads with progress indicators

Forms are the primary way users interact with your application. Master them, and you'll build secure, user-friendly web applications!