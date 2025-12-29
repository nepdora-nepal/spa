**Project Overview**
- **Stack:** Next.js 14 (App Router) + TypeScript + TailwindCSS.
- **Code layout:** main sources live under `src/` with the App Router in `src/app/`.
- **Data & state:** TanStack React Query for server state, many small custom hooks (`src/hooks/*`), and React Contexts for cross-cutting state (e.g., `src/contexts/CartContext.tsx`).

**Why this structure**
- The project separates UI (`src/components/`), domain hooks (`src/hooks/`), API clients (`src/services/`), and types (`src/types/`). This keeps components thin and moves side-effects into hooks/services.

**Key files and places to edit**
- `src/app/` — Next.js App Router pages/layouts; add routes and server components here.
- `src/components/` — UI primitives and composed components. Follow existing naming and export patterns.
- `src/hooks/` — Custom hooks for fetching and mutation. New data interactions should create a hook here that uses `@tanstack/react-query` and service functions in `src/services/` (see `use-product.ts` for an example).
- `src/services/` — API wrapper functions (e.g., `services/api/product`). Keep calls typed and return shapes matching `src/types/*`.
- `src/schemas/` — Zod schemas for forms. Use these to validate form data and infer types.
- `src/lib/` & `src/utils/` — shared helpers (jwt, strings, etc.).

**Data flow & patterns (concrete examples)**
- Hooks call service functions and expose `useQuery` / `useMutation` results. Example: `src/hooks/use-product.ts` imports `productApi` from `src/services/api/product` and returns `useQuery({ queryKey: ['products', params], queryFn: () => productApi.getProducts(params) })`.
- When a mutation succeeds, hooks invalidate relevant React Query keys (`queryClient.invalidateQueries({ queryKey: ['products'] })`). Follow this pattern to keep UI in sync.
- URL-driven filters: some hooks (e.g., `useProductFilters` in `use-product.ts`) read `useSearchParams()` and update router state via `next/navigation` — prefer this pattern for paginated/filterable lists.

**Conventions & small rules**
- **Icons:** ALWAYS use `lucide-react` only. Do not use other icon libraries unless explicitly requested.
- **UI Components:** ALWAYS prioritize existing Shadcn components from `src/components/ui/`. If a primitive is missing, check the reference below before building from scratch.
- **Styling:** Use TailwindCSS exclusively. Follow the existing design system (colors, spacing).
- **Types:** Prefer putting types in `src/types/*` and import with `@/...` aliases.
- **Form validation:** Use Zod files in `src/schemas/` — reuse and `z.infer` when possible.
- **Logic placement:** UI components are presentational; business logic belongs in hooks or `src/services/`.
- **Feedback:** Use `sonner` for toast notifications when signaling success/failure in hooks (existing pattern in `use-product.ts`).

**Build / dev / lint commands**
- The repo uses Node + `pnpm` (there is a `pnpm-lock.yaml`). Recommended local workflow:

```bash
pnpm install
pnpm dev      # runs `next dev`
pnpm build    # runs `next build`
pnpm start    # runs `next start` for production
pnpm lint     # runs `next lint`
```

If `pnpm` is not available, `npm`/`yarn` can run the same scripts but prefer `pnpm` to match the lockfile.

**Testing & missing pieces**
- There are no automated test scripts in `package.json` in this template. When adding tests, keep them co-located near code (e.g., `__tests__` next to hooks/services) and add test scripts to `package.json`.

**Integration points & external dependencies**
- React Query (`@tanstack/react-query`) is the primary data-sync tool; follow its caching/invalidation patterns.
- Zod is used for schema validation; use `@hookform/resolvers` + `react-hook-form` for controlled forms.
- Several Radix UI primitives and Tailwind-based styles are used for UI controls.

**How an AI agent should make edits**
- Prefer minimal, focused changes: add a new hook in `src/hooks/`, then the corresponding service call in `src/services/api/`, and finally update a page under `src/app/` that consumes the hook.
- When introducing new API keys or endpoints, add typed request/response shapes to `src/types/` and optional form/schema to `src/schemas`.
- Update `src/config/site.ts` for site-level settings rather than sprinkling literals across files.

**Code examples to follow (copy-ready patterns)**
- Query hook skeleton (see `src/hooks/use-product.ts`):

```ts
return useQuery({
  queryKey: ['resource', params],
  queryFn: () => api.resource.get(params),
  staleTime: 5 * 60_000,
});
```

- Mutation skeleton (invalidate queries on success):

```ts
return useMutation({
  mutationFn: data => api.resource.create(data),
  onSuccess: () => queryClient.invalidateQueries({ queryKey: ['resource'] }),
});
```

**When to ask a human**
- Adding or changing remote API base URLs, auth flows, or environment variables.
- Large refactors touching many modules — ask to confirm desired public API changes.

---

## Detailed Reference: Types, Services, & Hooks

This section provides a pseudocode reference for the repository's API surface. Use this to infer arguments, return types, and available methods.

### 1. Product & Commerce Domain

#### Types (`src/types/product.ts`, `orders.ts`, `cart.ts`, `pricing.ts`, `delivery.ts`)
```typescript
interface Product { id: number; name: string; slug: string; price: string; market_price: string; thumbnail_image: string; }
interface Category { id: number; name: string; slug: string; image?: string; }
interface SubCategory { id: number; name: string; slug: string; category: number; }
interface Order { id: number; order_number: string; status: string; total_amount: string; items: OrderItem[]; }
interface OrderItem { product_id: number; quantity: number; price: string; }
interface Pricing { id: number; name: string; price: string; features: PricingFeature[]; }
interface DeliveryCharge { location_name: string; default_cost: string; }
interface PaymentGateway { id: string; payment_type: "esewa" | "khalti"; is_enabled: boolean; }
interface PromoCode { code: string; discount_percentage: string; }
```

#### Services & Hooks
```typescript
// Product
const productApi = {
  getProducts(params: FilterParams): Promise<Page<Product>>,
  getProduct(slug: string): Promise<Product>,
  createProduct(data): Promise<Product>,
  updateProduct(slug, data): Promise<Product>,
  deleteProduct(slug): Promise<void>
}
const useProducts = (params) => UseQuery<Page<Product>>
const useProduct = (slug) => UseQuery<Product>
const useCreateProduct = () => UseMutation<Product>

// Order
const orderApi = { createOrder(data), getOrders(params), getOrder(id), updateStatus(id, status) }
const useCreateOrder = () => UseMutation<Order>
const useOrders = (params) => UseQuery<Page<Order>>

// Cart
const useCart = () => UseCartStore // (State management)

// Pricing & Delivery
const usePricings = () => UseQuery<Pricing[]>
const useDeliveryCharges = () => UseQuery<DeliveryCharge[]>
const usePaymentGateways = () => UseQuery<PaymentGateway[]>
```

### 2. Content & Media Domain

#### Types (`src/types/blog.ts`, `banner.ts`, `portfolio.ts`, `faq.ts`, `services.ts`)
```typescript
interface BlogPost { id: number; title: string; slug: string; content: string; tags: BlogTag[]; }
interface Banner { id: number; images: BannerImage[]; type: "Slider"|"Sidebar"; }
interface Portfolio { id: number; title: string; category: PortfolioCategory; images: string[]; }
interface ServicePost { id: number; title: string; description: string; }
interface FAQ { id: number; question: string; answer: string; }
interface Video { id: number; url: string; platform: string; }
interface Testimonial { id: number; name: string; comment: string; }
```

#### Services & Hooks
```typescript
// Blog
const blogApi = { getBlogs(params), getBlog(slug), getTags() }
const useBlogs = (params) => UseQuery<Page<BlogPost>>
const useBlog = (slug) => UseQuery<BlogPost>

// Banner
const bannerApi = { getBanners() }
const useBanners = () => UseQuery<Banner[]>

// Portfolio
const portfolioApi = { getPortfolios(params), getPortfolio(slug) }
const usePortfolios = (params) => UseQuery<Page<Portfolio>>

// Standard Content Hooks
const useServices = () => UseQuery<ServicePost[]>
const useFAQs = () => UseQuery<FAQ[]>
const useVideos = () => UseQuery<Video[]>
const useTestimonials = () => UseQuery<Testimonial[]>
```

### 3. System & Engagement Domain

#### Types (`src/types/site-config.ts`, `contact.ts`, `popup.ts`, `team.ts`, `newsletter.ts`)
```typescript
interface SiteConfig { business_name: string; phone: string; email: string; social_links: Socials; }
interface ContactSubmission { name: string; email: string; message: string; }
interface Popup { title: string; is_active: boolean; form_fields: Field[]; }
interface TeamMember { name: string; role: string; photo: string; }
interface NewsletterSub { email: string; }
```

#### Services & Hooks
```typescript
// Site Config
const siteConfigAPI = { getSiteConfig(), updateSiteConfig(data) }
const useSiteConfig = () => UseQuery<SiteConfig>

// Forms & Popups
const useSubmitContactForm = () => UseMutation<void> // calls contactAPI.submit()
const usePopups = () => UseQuery<Popup[]>
const useActivePopup = () => UseQuery<Popup> // filtered for active

// Team & Newsletter
const useTeamMembers = () => UseQuery<TeamMember[]>
const useCreateNewsletter = () => UseMutation<void>

// Integrations (GA, Facebook, WhatsApp)
const useGoogleAnalytics = () => UseQuery<GAConfig>
const useFacebookIntegration = () => UseQuery<FBConfig> // from facebook.ts
const useWhatsApps = () => UseQuery<WhatsAppConfig[]>
```

### 4. Booking & Support Domain

#### Types (`src/types/appointment.ts`, `booking.ts`, `issues.ts`)
```typescript
interface Appointment { id: number; date: string; status: string; full_name: string; }
interface Booking { id: number; data: BookingData; collection_id: number; }
interface Issue { id: number; title: string; status: "open"|"closed"; priority: string; }
```

#### Services & Hooks
```typescript
// Appointment
const useSubmitAppointmentForm = () => UseMutation<void>
const useGetAppointments = (params) => UseQuery<Page<Appointment>>

// Booking
const useGetBookings = (params) => UseQuery<Page<Booking>>

// Issues (Support Tickets)
const issuesApi = { getIssues(), createIssue(), updateIssue() }
// Note: Hooks likely follow standard pattern: useIssues, useCreateIssue
```

### 5. Authentication Domain

#### Types (`src/types/auth/customer/auth.ts`)
```typescript
interface User { id: number; email: string; first_name: string; last_name: string; phone?: string; address?: string; }
interface AuthTokens { access: string; refresh: string; }
interface DecodedAccessToken { exp: number; user_id: number; email: string; ... }
```

#### Services & Context
```typescript
// Auth API (src/services/auth/customer/api.ts)
const authApi = { loginUser(data), signupUser(data) }

// Auth Context (src/contexts/AuthContext.tsx)
// use useContext(AuthContext) to access:
// { user, tokens, login(data), signup(data), logout(), isLoading, isAuthenticated }
```

---

## Reference: Components, Utils, & Config

### 1. UI Components (`src/components/ui/`)
This project uses a Shadcn-like component library. Prefer these primitives over raw HTML/Tailwind.
- **Layout/Structure:** `card`, `sheet` (sidebar/drawer), `dialog` (modal), `accordion`, `separator`, `scroll-area`, `resizable`, `aspect-ratio`
- **Inputs/Forms:** `button`, `input`, `textarea`, `checkbox`, `radio-group`, `select`, `switch`, `slider`, `toggle`
- **Feedback:** `alert`, `badge`, `sonner` (toast), `progress`, `spinner`, `skeleton` (loading state)
- **Data Display:** `table`, `avatar`, `carousel`, `chart`, `calendar`
- **Navigation:** `navigation-menu`, `breadcrumb`, `pagination`, `tabs`, `dropdown-menu`, `menubar`

### 2. Utils (`src/utils/`, `src/lib/`)
- **Authentication:** `getAuthToken()`, `getAuthTokenCustomer()` (read from cookies/storage)
- **Networking:** `createHeaders()`, `createHeadersCustomer()` (attach tokens to requests), `handleApiError(response)` (unified error thrower)
- **Formatting:** `cn(...inputs)` (Tailwind class merger), `capitalizeWords(str)`
- **Validation:** `validateSocialUrls(data)`, `validateFile(file, maxSize)`, `validateUrl(url)`
- **Configuration:** `siteConfig` (base URLs, static assets pathing), `getApiBaseUrl()`, `getImageUrl(path)`

### 3. Contexts (`src/contexts/`)
- **CartContext:** Manages e-commerce cart state.
  - `useCart()` exposes: `{ cartItems, addToCart, removeFromCart, updateQuantity, clearCart, totalAmount }`
- **AuthContext:** Manages customer authentication (JWT-based).
  - Use `useContext(AuthContext)` to access: `{ user, tokens, login, signup, logout, isLoading, isAuthenticated }`
  - Wrap application with `CustomerPublishAuthProvider`.

### 4. Schemas (`src/schemas/`)
Zod definitions for form validation. Use these with `react-hook-form`.
- **Auth:** `login.form.ts`, `signup.form.ts`, `customer/login.form.ts`
- **Content:** `blog.form.ts`, `portfolio.form.ts`, `services.form.ts`
- **Commerce:** `product.form.ts`, `category.form.ts`, `checkout.form.ts`, `promocode.form.ts`
- **Others:** `issues.form.ts`

---

# CODEBASE MASTER INDEX (PSEUDOCODE REFERENCE)

## 1. TYPES (`src/types/*.ts`)

### `product.ts` (Core Commerce)
```typescript
interface Product {
  id: number;
  name: string;
  slug: string;
  price: string;
  market_price: string;
  description: string;
  thumbnail_image: string;
  thumbnail_alt_description: string;
  images: ProductImage[];
  variants: ProductVariant[];
  options: ProductOption[];
  category: Category;
  sub_category: SubCategory;
  is_featured: boolean;
  is_popular: boolean;
  in_stock: boolean;
  created_at: string;
}

interface ProductVariant {
  id: number;
  price: string;
  stock: number;
  image: string | null;
  option_values: VariantOptionValue[];
}

interface Category {
  id: number;
  name: string;
  slug: string;
  image?: string;
  description?: string;
}

interface SubCategory {
  id: number;
  name: string;
  slug: string;
  category: number;
}
```

### `orders.ts` & `cart.ts`
```typescript
interface Order {
  id: number;
  order_number: string;
  customer_name: string;
  customer_email: string;
  customer_phone: string;
  customer_address: string;
  shipping_address: string;
  city: string;
  total_amount: string;
  delivery_charge: string;
  discount_amount: string;
  status: "pending" | "confirmed" | "processing" | "shipped" | "delivered" | "cancelled";
  payment_type: string;
  is_paid: boolean;
  items: OrderItem[];
  created_at: string;
}

interface OrderItem {
  id: number;
  product_id: number;
  product: ProductBrief;
  variant_id?: number;
  variant?: VariantBrief;
  quantity: number;
  price: string;
}

interface CartItem {
  product: Product;
  quantity: number;
  selectedVariant?: ProductVariant | null;
}
```

### `blog.ts` & `portfolio.ts` & `services.ts`
```typescript
interface BlogPost {
  id: number;
  title: string;
  slug: string;
  author: Author | null;
  content: string;
  thumbnail_image: string | null;
  tags: BlogTag[];
  is_published: boolean;
  created_at: string;
}

interface Portfolio {
  id: number;
  title: string;
  slug: string;
  content: string;
  thumbnail_image: string | null;
  category: PortfolioCategory;
  tags: PortfolioTag[];
  project_url: string | null;
  github_url: string | null;
}

interface ServicesPost {
  id: number;
  title: string;
  slug: string;
  description: string;
  thumbnail_image: string | null;
}
```

### `auth.ts` (Customer Authentication)
```typescript
interface User {
  id: number;
  first_name: string;
  last_name: string;
  email: string;
  phone?: string | null;
  address?: string | null;
}

interface AuthTokens {
  access: string;
  refresh: string;
}

interface SignupResponse {
  id: number;
  first_name: string;
  last_name: string;
  email: string;
}
```

### `system.ts` & `config.ts`
```typescript
interface SiteConfig {
  id: number;
  business_name: string;
  business_description: string;
  logo: string | null;
  favicon: string | null;
  email: string;
  phone: string;
  address: string;
  social_links: {
    facebook: string;
    instagram: string;
    twitter: string;
    linkedin: string;
    youtube: string;
    tiktok: string;
  };
}

interface WhatsApp {
  id: string;
  phone_number: string;
  message: string;
  is_enabled: boolean;
}

interface GoogleAnalytics {
  id: string;
  measurement_id: string;
  is_enabled: boolean;
}
```

## 2. SERVICES (`src/services/api/*.ts`)

```typescript
// Product API (src/services/api/product.ts)
const productApi = {
  getProducts: (params?: ProductFilterParams) => Promise<GetProductsResponse>,
  getProduct: (slug: string) => Promise<Product>,
  createProduct: (data: CreateProductRequest) => Promise<CreateProductResponse>,
  updateProduct: (slug: string, data: UpdateProductRequest) => Promise<UpdateProductResponse>,
  deleteProduct: (slug: string) => Promise<DeleteProductResponse>,
}

// Order API (src/services/api/orders.ts)
const orderApi = {
  getOrders: (params: OrderPaginationParams) => Promise<OrdersResponse>,
  getOrder: (id: number) => Promise<Order>,
  createOrder: (data: CreateOrderRequest) => Promise<Order>,
  updateStatus: (id: number, statusData: UpdateOrderStatusRequest) => Promise<Order>,
  updateOrderPayment: (id: number, paymentData: UpdateOrderPaymentRequest) => Promise<Order>,
}

// Banner API (src/services/api/banner.ts)
const bannerApi = {
  getBanners: () => Promise<Banner[]>,
  getBanner: (id: number) => Promise<Banner>,
  create: (data: CreateBannerWithImagesRequest | FormData) => Promise<Banner>,
  update: (id: number, data: UpdateBannerWithImagesRequest | FormData) => Promise<Banner>,
  delete: (id: number) => Promise<void>,
}

// Blog API (src/services/api/blog.ts)
const blogApi = {
  getBlogs: (filters?: BlogFilters) => Promise<PaginatedBlogResponse>,
  getBlogBySlug: (slug: string) => Promise<BlogPost>,
  getBlogTags: () => Promise<BlogTag[]>,
  create: (data: CreateBlogPost) => Promise<BlogPost>,
  update: (slug: string, data: UpdateBlogPost) => Promise<BlogPost>,
  delete: (slug: string) => Promise<void>,
}

// FAQ API (src/services/api/faq.ts)
const faqApi = {
  getFAQs: () => Promise<FAQ[]>,
  getFAQ: (id: number) => Promise<FAQ>,
  createFAQ: (data: CreateFAQRequest) => Promise<FAQ>,
  updateFAQ: (id: number, data: UpdateFAQRequest) => Promise<FAQ>,
  deleteFAQ: (id: number) => Promise<void>,
}

// Testimonials API (src/services/api/testimonials.ts)
const testimonialsApi = {
  getTestimonials: () => Promise<Testimonial[]>,
  getTestimonial: (id: number) => Promise<Testimonial>,
  create: (data: CreateTestimonialData) => Promise<Testimonial>,
  update: (id: number, data: UpdateTestimonialData) => Promise<Testimonial>,
  delete: (id: number) => Promise<void>,
}

// Team API (src/services/api/team-member.ts)
const teamAPI = {
  getTeams: () => Promise<Members>,
  getTeam: (id: number) => Promise<TEAM>,
  createTeam: (data: FormData) => Promise<TEAM>,
  updateTeam: (id: number, data: FormData) => Promise<TEAM>,
  deleteTeam: (id: number) => Promise<void>,
}

// Contact API (src/services/api/contact.ts)
const contactAPI = {
  getContacts: (filters?: ContactFilters) => Promise<PaginatedContacts>,
  submitContactForm: (data: ContactFormData) => Promise<void>,
}

// Newsletter API (src/services/api/newsletter.ts)
const newsletterApi = {
  getNewsletters: (page: number, pageSize: number, search: string) => Promise<any>,
  createNewsletter: (data: { email: string }) => Promise<void>,
}

// Popup API (src/services/api/popup.ts)
const popupApi = {
  getPopups: () => Promise<PopUp[]>,
  getPopup: (id: number) => Promise<PopUp>,
  getActivePopup: () => Promise<PopUp>,
  createPopup: (data: FormData) => Promise<PopUp>,
  updatePopup: (id: number, data: FormData) => Promise<PopUp>,
  deletePopup: (id: number) => Promise<void>,
  getPopupForms: (filters?: PopupFormFilters) => Promise<PaginatedPopupFormResponse>,
  submitPopupForm: (data: PopupFormData) => Promise<void>,
}

// Portfolio API (src/services/api/portfolio.ts)
const portfolioApi = {
  getPortfolios: (filters?: PortfolioFilters) => Promise<PaginatedPortfolioResponse>,
  getPortfolioBySlug: (slug: string) => Promise<Portfolio>,
  getPortfolioCategories: () => Promise<PortfolioCategory[]>,
  getPortfolioTags: () => Promise<PortfolioTag[]>,
  create: (data: CreatePortfolio) => Promise<Portfolio>,
  update: (slug: string, data: UpdatePortfolio) => Promise<Portfolio>,
  delete: (slug: string) => Promise<void>,
}

// Services API (src/services/api/services.ts)
const servicesApi = {
  getServices: (filters?: ServicesFilters) => Promise<PaginatedServicesResponse>,
  getServiceBySlug: (slug: string) => Promise<ServicesPost>,
  create: (data: CreateServicesPost) => Promise<ServicesPost>,
  update: (slug: string, data: UpdateServicesPost) => Promise<ServicesPost>,
  delete: (slug: string) => Promise<void>,
}

// Pricing API (src/services/api/pricing.ts)
const usePricingApi = {
  getPricings: (params?: PricingQueryParams) => Promise<GetPricingsResponse>,
  getPricing: (id: number) => Promise<Pricing>,
  createPricing: (data: CreatePricingRequest) => Promise<CreatePricingResponse>,
  updatePricing: (id: number, data: UpdatePricingRequest) => Promise<UpdatePricingResponse>,
  deletePricing: (id: number) => Promise<DeletePricingResponse>,
}

// Auth API (src/services/auth/customer/api.ts)
const authApi = {
  loginUser: (data: LoginData) => Promise<LoginResponse>,
  signupUser: (data: SignupData) => Promise<SignupResponse>,
}
```

## 3. HOOKS (`src/hooks/*.ts`)

### Commerce Hooks
```typescript
// use-product.ts
useProducts(additionalParams: ProductFilterParams) // Returns useQuery result
useProduct(slug: string) // Returns useQuery result
useCreateProduct() // Returns useMutation result
useUpdateProduct() // Returns useMutation result with mutate({ slug, data })
useDeleteProduct() // Returns useMutation result
useProductFilters() // Returns ProductFilterParams from URL
useUpdateFilters() // Returns { updateFilters, clearFilters }

// use-category.ts / use-subcategory.ts
useCategories(params: PaginationParams)
useCategory(slug: string)
useCreateCategory()
useUpdateCategory()
useDeleteCategory()
useSubCategories(params: PaginationParams)
useSubCategory(slug: string)

// use-cart.ts
useCart() // { cartItems, addToCart, removeFromCart, updateQuantity, clearCart, totalAmount }

// use-orders.ts
useOrders(params: OrderPaginationParams)
useOrder(id: number)
useCreateOrder()
useUpdateOrderStatus()

// use-promocode.ts / use-promo-code-validate.ts
usePromoCodes(params: PaginationParams)
usePromoCode(id: number)
useCreatePromoCode()
useUpdatePromoCode()
useDeletePromoCode()
useValidatePromoCode() // { mutate: validatePromoCode, isLoading, error, data }
```

### Content Hooks
```typescript
// use-blogs.ts
useBlogs(filters: BlogFilters)
useBlog(slug: string)
useBlogTags()
useCreateBlog()
useUpdateBlog()
useDeleteBlog()

// use-banner.ts
useBanners()
useBanner(id: number)
useCreateBannerWithImages()
useUpdateBannerWithImages()
useDeleteBanner()

// use-portfolio.ts
usePortfolios(filters: PortfolioFilters)
usePortfolio(slug: string)
usePortfolioCategories()
usePortfolioTags()
useCreatePortfolio()
useUpdatePortfolio()
useDeletePortfolio()

// use-services.ts
useServices(filters: ServicesFilters)
useService(slug: string)
useCreateService()
useUpdateService()
useDeleteService()

// use-faq.ts
useFAQs()
useFAQ(id: number)
useCreateFAQ()
useUpdateFAQ()
useDeleteFAQ()

// use-testimonials.ts
useTestimonials()
useTestimonial(id: number)
useCreateTestimonial()
useUpdateTestimonial()
useDeleteTestimonial()

// use-videos.ts
useVideos()
useCreateVideo()
useUpdateVideo()
useDeleteVideo()
```

### System & Booking Hooks
```typescript
// use-site-config.ts
useSiteConfig()
useCreateSiteConfig()
usePatchSiteConfig()
useDeleteSiteConfig()

// use-google-analytics.ts
useGoogleAnalytics()
useCreateGoogleAnalytics()
useUpdateGoogleAnalytics()
useDeleteGoogleAnalytics()

// use-whatsapp.ts
useWhatsApps()
useCreateWhatsApp()
useUpdateWhatsApp()
useDeleteWhatsApp()

// use-payment-gateway.ts
usePaymentGateways()
useCreatePaymentGateway()
useUpdatePaymentGateway()
useDeletePaymentGateway()

// use-appointment.ts
useGetAppointments(filters: AppointmentFilters)
useGetAppointmentReasons()
useSubmitAppointmentForm(siteUser: string)
useUpdateAppointment()
useDeleteAppointment()

// use-booking.ts
useGetBookings(filters: BookingFilters)
useGetBooking(id: number)
useUpdateBooking()

// use-mobile.tsx
useIsMobile() // Returns boolean
```

## 4. COMPONENTS & UTILS

### `src/components/common/ImageWithFallback.tsx`
```typescript
interface ImageWithFallbackProps extends ImageProps {
  fallbackSrc?: string;
  alt: string;
  imageId?: string; // Used for AI-driven dynamic replacement
}
```

### `src/lib/utils.ts` & `src/utils/*.ts`
```typescript
// Utils
cn(...inputs): string // Class merging (Tailwind)
getAuthToken(): string // Fetches JWT
createHeaders(): Headers // Prepared with Auth token
handleApiError(response): void // Handles 4xx/5xx responses centrally

// Media Helpers (src/lib/video-utils.ts)
extractVideoInfo(url): { id, platform, thumbnail }
getVideoEmbedUrl(url): string
```

### `src/config/site.ts`
```typescript
const siteConfig = {
  name: "Site Name",
  description: "Site Description",
  apiBaseUrl: process.env.NEXT_PUBLIC_API_URL,
  // ...other static configurations
}
```

## 5. APP STRUCTURE (`src/app/`)
- `layout.tsx`: Root layout with `Providers` (Query, Cart, Theme).
- `page.tsx`: Main landing page entry.
- `globals.css`: Global Tailwind styles.

---
**END OF MASTER INDEX**

