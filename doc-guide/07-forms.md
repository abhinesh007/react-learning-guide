# 07 — Forms & Validation in React

> Comprehensive guide to managing forms in React: native controlled vs. uncontrolled approaches, manual validation, React Hook Form, Zod schema validation, Formik, and file uploads.

---

## Table of Contents

- [Controlled Components](#controlled-components)
  - [Core Concept](#core-concept)
  - [Handling Multiple Inputs Dynamically](#handling-multiple-inputs-dynamically)
  - [Complete Controlled Form Example](#complete-controlled-form-example)
- [Uncontrolled Components](#uncontrolled-components)
  - [Accessing Values with `useRef`](#accessing-values-with-useref)
  - [`defaultValue` vs. `value`](#defaultvalue-vs-value)
  - [When to Use Uncontrolled Components](#when-to-use-uncontrolled-components)
- [Manual Form Validation](#manual-form-validation)
  - [Validation Strategies: Blur, Change, Submit](#validation-strategies-blur-change-submit)
  - [Error State Management & Accessibility](#error-state-management--accessibility)
- [React Hook Form (Industry Standard)](#react-hook-form-industry-standard)
  - [Why React Hook Form Dominates](#why-react-hook-form-dominates)
  - [Core API: `useForm`, `register`, `handleSubmit`](#core-api-useform-register-handlesubmit)
  - [Built-in Validation Rules & Error Messages](#built-in-validation-rules--error-messages)
  - [Interactive Helpers: `watch`, `setValue`, `reset`](#interactive-helpers-watch-setvalue-reset)
  - [Multi-Step Form Wizard Pattern](#multi-step-form-wizard-pattern)
- [Schema Validation with Zod + React Hook Form](#schema-validation-with-zod--react-hook-form)
  - [Why Schema-Based Validation?](#why-schema-based-validation)
  - [Setting up `@hookform/resolvers/zod`](#setting-up-hookformresolverszod)
  - [Full Type-Safe Registration Example](#full-type-safe-registration-example)
- [Formik + Yup (Alternative Ecosystem)](#formik--yup-alternative-ecosystem)
  - [Formik Fundamentals](#formik-fundamentals)
  - [Formik vs. React Hook Form](#formik-vs-react-hook-form)
- [File Uploads & Previews](#file-uploads--previews)
  - [Single & Multi-File Handlers](#single--multi-file-handlers)
  - [Client-Side Image Preview](#client-side-image-preview)
- [Form Solutions Comparison Table](#form-solutions-comparison-table)
- [🏋️ Practice Exercises](#️-practice-exercises)

---

## Controlled Components

### Core Concept

A **Controlled Component** binds its input value directly to React state. The input does not maintain its own internal state in the DOM; React serves as the "single source of truth":

```jsx
const [email, setEmail] = useState('');

<input
  value={email} // Driven by React state
  onChange={(e) => setEmail(e.target.value)} // React updates state
/>
```

---

### Handling Multiple Inputs Dynamically

Instead of writing 10 separate state variables and 10 separate `onChange` handlers for a 10-field form, use a single state object with a dynamic computed property name:

```jsx
import { useState } from 'react';

export function RegistrationForm() {
  const [formData, setFormData] = useState({
    firstName: '',
    lastName: '',
    role: 'developer',
    newsletter: false,
    bio: '',
  });

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value,
    }));
  };

  return (
    <form>
      <input
        name="firstName"
        placeholder="First Name"
        value={formData.firstName}
        onChange={handleChange}
      />
      <input
        name="lastName"
        placeholder="Last Name"
        value={formData.lastName}
        onChange={handleChange}
      />
      <select name="role" value={formData.role} onChange={handleChange}>
        <option value="developer">Developer</option>
        <option value="designer">Designer</option>
        <option value="manager">Product Manager</option>
      </select>
      <label>
        <input
          type="checkbox"
          name="newsletter"
          checked={formData.newsletter}
          onChange={handleChange}
        />
        Subscribe to updates
      </label>
      <textarea
        name="bio"
        value={formData.bio}
        onChange={handleChange}
      />
    </form>
  );
}
```

---

### Complete Controlled Form Example

```jsx
import { useState } from 'react';

export function ContactForm() {
  const [formData, setFormData] = useState({ name: '', email: '', message: '' });
  const [status, setStatus] = useState('idle'); // 'idle' | 'submitting' | 'success' | 'error'

  const handleSubmit = async (e) => {
    e.preventDefault();
    setStatus('submitting');

    try {
      const res = await fetch('/api/contact', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData),
      });
      if (!res.ok) throw new Error('Submission failed');
      setStatus('success');
      setFormData({ name: '', email: '', message: '' }); // Reset form
    } catch (err) {
      setStatus('error');
    }
  };

  return (
    <form onSubmit={handleSubmit} className="contact-form">
      <div>
        <label htmlFor="name">Name:</label>
        <input
          id="name"
          name="name"
          required
          value={formData.name}
          onChange={e => setFormData({ ...formData, name: e.target.value })}
        />
      </div>

      <div>
        <label htmlFor="email">Email:</label>
        <input
          id="email"
          name="email"
          type="email"
          required
          value={formData.email}
          onChange={e => setFormData({ ...formData, email: e.target.value })}
        />
      </div>

      <div>
        <label htmlFor="message">Message:</label>
        <textarea
          id="message"
          name="message"
          required
          value={formData.message}
          onChange={e => setFormData({ ...formData, message: e.target.value })}
        />
      </div>

      <button type="submit" disabled={status === 'submitting'}>
        {status === 'submitting' ? 'Sending...' : 'Send Message'}
      </button>

      {status === 'success' && <p className="success-msg">Message sent successfully!</p>}
      {status === 'error' && <p className="error-msg">Failed to send message. Please retry.</p>}
    </form>
  );
}
```

---

## Uncontrolled Components

### Accessing Values with `useRef`

An **Uncontrolled Component** leaves form data management to the DOM itself. You pull values imperatively when the form is submitted using `useRef` or the standard `FormData` browser API:

```jsx
import { useRef } from 'react';

export function UncontrolledForm() {
  const nameRef = useRef(null);
  const emailRef = useRef(null);

  const handleSubmit = (e) => {
    e.preventDefault();
    const payload = {
      name: nameRef.current.value,
      email: emailRef.current.value,
    };
    console.log('Submitted Payload:', payload);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={nameRef} defaultValue="John Doe" type="text" />
      <input ref={emailRef} defaultValue="john@example.com" type="email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

### `defaultValue` vs. `value`

- When using controlled inputs, you provide `value` and must pair it with `onChange`.
- When using uncontrolled inputs, providing `value` makes the input read-only! Instead, use `defaultValue` (or `defaultChecked` for checkboxes) to specify the starting value without locking the DOM input.

---

### When to Use Uncontrolled Components

1. **File inputs (`<input type="file" />`)**: A file input is always uncontrolled in the DOM because its value cannot be programmatically set for security reasons.
2. **Integrating with non-React DOM libraries** (e.g., legacy jQuery plugins, rich text editors).
3. **High-performance massive data grids**: Eliminates re-renders on every keystroke.

---

## Manual Form Validation

### Validation Strategies: Blur, Change, Submit

- **On Submit**: Validates only when the user presses submit. Lowest friction, but can be annoying if errors are found late.
- **On Blur (`onBlur`)**: Validates when the user tabs away from a field. Best user experience (avoids yelling at users while they are still typing).
- **On Change (`onChange`)**: Validates on every keystroke. Great for password strength meters, but jarring for email formats before typing is complete.

---

### Error State Management & Accessibility

```jsx
import { useState } from 'react';

export function ValidatedInputForm() {
  const [email, setEmail] = useState('');
  const [touched, setTouched] = useState(false);
  const [error, setError] = useState('');

  const validate = (val) => {
    if (!val) return 'Email is required';
    if (!/\S+@\S+\.\S+/.test(val)) return 'Invalid email format';
    return '';
  };

  const handleBlur = () => {
    setTouched(true);
    setError(validate(email));
  };

  const handleChange = (e) => {
    setEmail(e.target.value);
    if (touched) {
      setError(validate(e.target.value));
    }
  };

  return (
    <div>
      <label htmlFor="user-email">Email Address</label>
      <input
        id="user-email"
        type="email"
        value={email}
        onChange={handleChange}
        onBlur={handleBlur}
        aria-invalid={touched && Boolean(error)}
        aria-describedby={error ? 'email-error' : undefined}
      />
      {touched && error && (
        <span id="email-error" role="alert" style={{ color: 'red', display: 'block' }}>
          {error}
        </span>
      )}
    </div>
  );
}
```

---

## React Hook Form (Industry Standard)

### Why React Hook Form Dominates

1. **Zero Unnecessary Re-renders**: RHF uses uncontrolled inputs under the hood via refs, subscribing only the elements that need to re-render.
2. **Minimal Boilerplate**: Removes verbose `value` + `onChange` code.
3. **Tiny Footprint**: Only ~8KB with zero dependencies.
4. **Seamless Integration**: Integrates directly with schema validation libraries (Zod, Yup, Joi).

```bash
npm install react-hook-form
```

---

### Core API: `useForm`, `register`, `handleSubmit`

```jsx
import { useForm } from 'react-hook-form';

export function LoginFormRHF() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = async (data) => {
    console.log('Validated Form Data:', data);
    await new Promise(resolve => setTimeout(resolve, 1000));
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label>Email</label>
        <input
          {...register('email', {
            required: 'Email is required',
            pattern: {
              value: /^[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}$/i,
              message: 'Invalid email address',
            },
          })}
        />
        {errors.email && <p className="error">{errors.email.message}</p>}
      </div>

      <div>
        <label>Password</label>
        <input
          type="password"
          {...register('password', {
            required: 'Password is required',
            minLength: {
              value: 8,
              message: 'Password must be at least 8 characters',
            },
          })}
        />
        {errors.password && <p className="error">{errors.password.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Authenticating...' : 'Sign In'}
      </button>
    </form>
  );
}
```

---

### Interactive Helpers: `watch`, `setValue`, `reset`

```jsx
import { useForm } from 'react-hook-form';

export function ProfileForm() {
  const { register, watch, setValue, reset, handleSubmit } = useForm({
    defaultValues: { country: 'US', zipCode: '' },
  });

  // Watch a single field reactively
  const selectedCountry = watch('country');

  const autofillZip = () => {
    setValue('zipCode', '94103', { shouldValidate: true, shouldDirty: true });
  };

  return (
    <form onSubmit={handleSubmit(data => console.log(data))}>
      <select {...register('country')}>
        <option value="US">United States</option>
        <option value="CA">Canada</option>
        <option value="UK">United Kingdom</option>
      </select>

      {/* Dynamic condition based on watched value */}
      {selectedCountry === 'US' && (
        <div>
          <label>US ZIP Code</label>
          <input {...register('zipCode')} />
          <button type="button" onClick={autofillZip}>Autofill SF Zip</button>
        </div>
      )}

      <button type="button" onClick={() => reset()}>Reset to Defaults</button>
      <button type="submit">Save</button>
    </form>
  );
}
```

---

### Multi-Step Form Wizard Pattern

```jsx
import { useState } from 'react';
import { useForm } from 'react-hook-form';

export function CheckoutWizard() {
  const [step, setStep] = useState(1);
  const { register, handleSubmit, trigger, formState: { errors } } = useForm();

  const handleNext = async (fieldsToValidate) => {
    // Only proceed to next step if current step's fields pass validation
    const isValid = await trigger(fieldsToValidate);
    if (isValid) setStep(s => s + 1);
  };

  const handleFinalSubmit = (data) => {
    console.log('Complete order submitted:', data);
  };

  return (
    <form onSubmit={handleSubmit(handleFinalSubmit)}>
      {/* STEP 1: Personal Info */}
      {step === 1 && (
        <section>
          <h3>Step 1: Contact Details</h3>
          <input {...register('fullName', { required: 'Name is required' })} placeholder="Full Name" />
          {errors.fullName && <p>{errors.fullName.message}</p>}

          <input {...register('email', { required: 'Email is required' })} placeholder="Email" />
          {errors.email && <p>{errors.email.message}</p>}

          <button type="button" onClick={() => handleNext(['fullName', 'email'])}>Next →</button>
        </section>
      )}

      {/* STEP 2: Shipping */}
      {step === 2 && (
        <section>
          <h3>Step 2: Shipping Address</h3>
          <input {...register('address', { required: 'Address is required' })} placeholder="Street Address" />
          <input {...register('city', { required: 'City is required' })} placeholder="City" />

          <button type="button" onClick={() => setStep(s => s - 1)}>← Back</button>
          <button type="button" onClick={() => handleNext(['address', 'city'])}>Next →</button>
        </section>
      )}

      {/* STEP 3: Payment Confirmation */}
      {step === 3 && (
        <section>
          <h3>Step 3: Review & Complete</h3>
          <button type="button" onClick={() => setStep(s => s - 1)}>← Back</button>
          <button type="submit">Place Order</button>
        </section>
      )}
    </form>
  );
}
```

---

## Schema Validation with Zod + React Hook Form

### Why Schema-Based Validation?

- **Single Source of Truth**: Centralize validation rules in declarative schemas.
- **Full TypeScript Inference**: Infer TS types directly from your Zod schema using `z.infer<typeof schema>`.
- **Reusable**: Share the exact same validation schemas between your React front-end and your Node/Express backend!

```bash
npm install zod @hookform/resolvers
```

---

### Full Type-Safe Registration Example

```jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// 1. Define Zod validation schema
const registrationSchema = z.object({
  username: z
    .string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username cannot exceed 20 characters'),
  email: z
    .string()
    .email('Please enter a valid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Must contain at least 1 uppercase letter')
    .regex(/[0-9]/, 'Must contain at least 1 number'),
  confirmPassword: z.string(),
  acceptTerms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the Terms of Service' }),
  }),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'], // Anchor error to confirmPassword field
});

export function SignUpForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm({
    resolver: zodResolver(registrationSchema),
    mode: 'onBlur', // Validate on field blur
  });

  const onSubmit = async (data) => {
    const res = await fetch('/api/register', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label>Username</label>
        <input {...register('username')} />
        {errors.username && <p className="err">{errors.username.message}</p>}
      </div>

      <div>
        <label>Email</label>
        <input type="email" {...register('email')} />
        {errors.email && <p className="err">{errors.email.message}</p>}
      </div>

      <div>
        <label>Password</label>
        <input type="password" {...register('password')} />
        {errors.password && <p className="err">{errors.password.message}</p>}
      </div>

      <div>
        <label>Confirm Password</label>
        <input type="password" {...register('confirmPassword')} />
        {errors.confirmPassword && <p className="err">{errors.confirmPassword.message}</p>}
      </div>

      <div>
        <label>
          <input type="checkbox" {...register('acceptTerms')} />
          I agree to the Terms of Service
        </label>
        {errors.acceptTerms && <p className="err">{errors.acceptTerms.message}</p>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating Account...' : 'Register'}
      </button>
    </form>
  );
}
```

---

## Formik + Yup (Alternative Ecosystem)

### Formik Fundamentals

Formik is the long-standing form library created by Jared Palmer. It uses controlled components via context:

```bash
npm install formik yup
```

```jsx
import { Formik, Form, Field, ErrorMessage } from 'formik';
import * as Yup from 'yup';

const validationSchema = Yup.object({
  email: Yup.string().email('Invalid email').required('Required'),
  password: Yup.string().min(6, 'Too short').required('Required'),
});

export function FormikLoginForm() {
  return (
    <Formik
      initialValues={{ email: '', password: '' }}
      validationSchema={validationSchema}
      onSubmit={(values) => console.log(values)}
    >
      {({ isSubmitting }) => (
        <Form>
          <Field type="email" name="email" placeholder="Email" />
          <ErrorMessage name="email" component="div" className="err" />

          <Field type="password" name="password" placeholder="Password" />
          <ErrorMessage name="password" component="div" className="err" />

          <button type="submit" disabled={isSubmitting}>Log In</button>
        </Form>
      )}
    </Formik>
  );
}
```

---

### Formik vs. React Hook Form

| Feature | React Hook Form | Formik |
|:---|:---|:---|
| **Under the Hood** | Uncontrolled (Refs) | Controlled (State) |
| **Re-render Frequency** | Minimal (isolated) | Re-renders on every keystroke across entire form |
| **Package Size** | ~8 KB | ~15 KB |
| **Ecosystem Momentum** | Highly Active / Industry Standard | Lower activity / Maintenance mode |
| **Best For** | New projects, performance-critical forms | Existing legacy codebases |

---

## File Uploads & Previews

### Single & Multi-File Handlers

```jsx
import { useState } from 'react';

export function FileUploader() {
  const [file, setFile] = useState(null);
  const [previewUrl, setPreviewUrl] = useState(null);
  const [uploadProgress, setUploadProgress] = useState(0);

  const handleFileChange = (e) => {
    const selected = e.target.files?.[0];
    if (selected) {
      setFile(selected);
      // Generate client-side preview URL
      setPreviewUrl(URL.createObjectURL(selected));
    }
  };

  const handleUpload = async () => {
    if (!file) return;

    const formData = new FormData();
    formData.append('avatar', file);

    const xhr = new XMLHttpRequest();
    xhr.open('POST', '/api/upload');

    xhr.upload.onprogress = (event) => {
      if (event.lengthComputable) {
        setUploadProgress(Math.round((event.loaded / event.total) * 100));
      }
    };

    xhr.onload = () => {
      if (xhr.status === 200) alert('Upload complete!');
    };

    xhr.send(formData);
  };

  return (
    <div>
      <input type="file" accept="image/*" onChange={handleFileChange} />

      {previewUrl && (
        <div style={{ marginTop: 12 }}>
          <img src={previewUrl} alt="Preview" style={{ width: 120, height: 120, objectFit: 'cover' }} />
        </div>
      )}

      {file && (
        <div>
          <p>{file.name} ({(file.size / 1024).toFixed(1)} KB)</p>
          <button type="button" onClick={handleUpload}>Upload Avatar</button>
          {uploadProgress > 0 && <progress value={uploadProgress} max="100" />}
        </div>
      )}
    </div>
  );
}
```

---

## Form Solutions Comparison Table

| Feature | Native Controlled | React Hook Form | Formik |
|:---|:---|:---|:---|
| **External Dependencies** | None | 1 (`react-hook-form`) | 2 (`formik`, `yup`) |
| **Keystroke Performance** | Slow in large forms | Optimal (uncontrolled) | Slower in large forms |
| **TypeScript Support** | Manual | Native & First-Class | Good |
| **Schema Integration** | Custom code | Seamless (Zod, Yup) | Yup native |
| **Learning Curve** | Low | Low-Moderate | Low |
| **Modern Recommendation** | Simple 1-2 input forms | Default choice for 95% of forms | Legacy apps |

---

## 🏋️ Practice Exercises

### Exercise 1: Registration with RHF + Zod
Build a complete registration form with:
- `fullName`, `email`, `password`, `confirmPassword`, `birthDate` (must be 18+ years old)
- Zod schema validation with clear error messages under each field.

### Exercise 2: Multi-step Checkout Flow
Build a 3-step checkout wizard using React Hook Form:
- Step 1: Shipping address
- Step 2: Payment info
- Step 3: Review order summary & final submit

---

> **Previous:** [← 06 — Routing](./06-routing.md)  
> **Next:** [08 — Data Fetching →](./08-data-fetching.md)
