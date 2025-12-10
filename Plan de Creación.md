# Plan de Creación - Plataforma de Contenido con Validación Manual de Pagos

## 🎯 **Visión del Proyecto**
Plataforma de e-commerce simplificada donde los usuarios compran contenido digital (cursos, videos, documentos) mediante un flujo de pago manual: el usuario agrega productos al carrito, sube comprobante de pago, y el administrador valida y desbloquea el contenido específicamente para ese usuario.

---

## 📋 **Resumen de Requisitos Clave**

### **Características Eliminadas vs Proyecto Actual**
- ❌ **Sin integración Mercado Libre/Pago** - No webhooks externos
- ❌ **Sin sistema de envíos** - No ME2, no tracking, no cálculo de shipping
- ❌ **Sin checkout automatizado** - No pasarelas de pago automáticas
- ❌ **Sin sincronización de productos** - Gestión 100% local

### **Características Nuevas Implementadas**
- ✅ **Productos multimedia** - 1 imagen principal + 3 secundarias + videos
- ✅ **Sistema de ofertas** - Precios con descuentos visibles
- ✅ **Flujo de pago manual** - Subida de comprobantes
- ✅ **Validación administrativa** - Panel para aprobar/rechazar pagos
- ✅ **Desbloqueo por usuario** - Contenido específico por usuario-producto
- ✅ **Protección de contenido** - Blur en preview, anti-captura, watermarks
- ✅ **Acceso restringido** - Solo desde "Mis Compras" después de pago validado

---

## 🏗️ **Arquitectura y Stack Tecnológico**

### **Tecnologías Reutilizadas del Proyecto Base**
```
✅ Next.js 15.5 + React 19          # Framework principal
✅ TypeScript + Tailwind CSS 4.1    # Desarrollo y estilos
✅ NextAuth.js v5                   # Autenticación (sin OAuth ML)
✅ Drizzle ORM + PostgreSQL         # Base de datos
✅ Zustand                          # Estado global del carrito
✅ Componentes UI (shadcn/ui)       # Interfaz reutilizable
✅ Estructura de carpetas base      # Organización del proyecto
```

### **Componentes Eliminados Completamente**
```
❌ lib/mercado-libre/               # Todo el ecosistema ML
❌ lib/mercado-pago/                # SDK y webhooks MP
❌ lib/mercado-envios/              # Cálculo de envíos ME2
❌ app/api/webhooks/                # Endpoints externos
❌ Integraciones MCP                 # Servers ML/MP
❌ Migraciones de sincronización     # Scripts ML
```

### **Nuevos Módulos a Construir**
```
🆕 lib/payment-proofs/              # Gestión de comprobantes
🆕 lib/content-protection/          # Sistema anti-captura y watermarks
🆕 lib/content-unlock/              # Sistema de desbloqueo
🆕 components/media-gallery/        # Galería de imágenes/videos
🆕 components/payment-proof/        # Subida y vista de comprobantes
🆕 app/admin/payments/              # Panel de validación
```

---

## 🗄️ **Modelo de Datos Simplificado**

### **Tablas Principales (Reutilizadas)**
```sql
-- Usuarios y autenticación
users (id, email, name, role, created_at)
categories (id, name, description, created_at)

-- Productos (MODIFICADA)
products (
  id, title, description, price, sale_price, 
  category_id, status, created_at, updated_at,
  -- Campos multimedia
  main_image_url, 
  secondary_images_1, secondary_images_2, secondary_images_3,
  video_url, video_thumbnail
)

-- Carrito (REUTILIZADA)
carts (id, user_id, created_at, updated_at)
cart_items (id, cart_id, product_id, quantity, price_snapshot)

-- Órdenes simplificadas (MODIFICADA)
orders (
  id, user_id, total_amount, status, 
  created_at, updated_at,
  -- Nuevos campos
  payment_status, -- pending/approved/rejected
  admin_notes,    -- Notas del administrador
  proof_image_url, -- URL del comprobante
  proof_text      -- Texto del comprobante
)
```

### **Nuevas Tablas Específicas**
```sql
-- Comprobantes de pago
payment_proofs (
  id, order_id, user_id,
  image_url,           -- Imagen del comprobante
  proof_text,          -- Texto descriptivo del pago
  status,              -- pending/approved/rejected
  admin_notes,         -- Notas de validación
  validated_by,        -- ID del admin que validó
  validated_at,        -- Timestamp de validación
  created_at
)

-- Contenido desbloqueado por usuario
user_unlocked_content (
  id, user_id, product_id, order_id,
  unlocked_at,         -- Cuando se desbloqueó
  unlocked_by,         -- Admin que lo desbloqueó
  access_expires_at,   -- Opcional: expiración del acceso
  created_at
)

-- Media de productos (imágenes/videos)
product_media (
  id, product_id, media_type, -- image/video
  url, thumbnail_url, alt_text,
  sort_order, is_main, created_at
)
```

### **Relaciones Clave**
```
users → orders → payment_proofs → user_unlocked_content
products → product_media (1:N)
orders → order_items → products
```

---

## 🔄 **Flujo de Usuario Completo**

### **1. Navegación y Selección (Contenido Protegido)**
```
Homepage → Catálogo → Vista Producto
    ↓           ↓           ↓
Categorías  Filtros   Galería multimedia
            Búsqueda  Precio/Oferta
                      Videos demo
                      🔒 CONTENIDO CON BLUR
```

### **2. Carrito de Compras**
```
Vista Producto → "Agregar al Carrito" → Carrito
                     ↓                      ↓
              Validación stock      Resumen de compra
              Precio con oferta     Subtotal calculado
                                       ↓
                                  "Proceder al Pago"
```

### **3. Proceso de Pago Manual**
```
Carrito → Checkout Manual → Formulario Comprobante
   ↓           ↓                    ↓
Resumen  Instrucciones de    Subir imagen del
Final   pago (transferencia) comprobante + texto
   ↓           ↓                    ↓
Confirmar  Generar orden      Validación frontend
Orden     con status          de archivo/tamaño
          "payment_pending"
```

### **4. Esperando Validación**
```
Checkout → Panel Usuario → "Mis Compras"
    ↓           ↓              ↓
Confirmación Lista de órdenes Estado:
de orden    con estados     ✅ Pagado y desbloqueado
            🟡 Esperando   ❌ Pago rechazado
            validación     📝 En revisión
```

### **5. Desbloqueo de Contenido (Acceso Restringido)**
```
Admin valida → Notificación usuario → Acceso contenido
      ↓               ↓                    ↓
Cambio status Email/SMS    "Mis Compras" → Vista producto
a "approved" automático    (ÚNICO ACCESO)
                          🔒 CONTENIDO COMPLETO
                          🛡️ ANTI-CAPTURA ACTIVO
                          💧 WATERMARK USUARIO
```

---

## 🛡️ **Sistema de Protección de Contenido**

### **🔒 Niveles de Seguridad Implementados**

#### **Nivel 1: Preview Protegido (Blur)**
```css
/* Contenido no comprado */
.content-protected {
  filter: blur(12px);
  user-select: none;
  pointer-events: none;
  -webkit-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
}

/* Overlay de protección */
.protection-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.1);
  z-index: 10;
}
```

#### **Nivel 2: Anti-Captura Activo**
```typescript
// components/content-protection/AntiCaptureProvider.tsx
const AntiCaptureProvider = ({ children, isUnlocked }: Props) => {
  useEffect(() => {
    if (!isUnlocked) return;

    // Bloquear clic derecho
    const preventContextMenu = (e: MouseEvent) => e.preventDefault();
    document.addEventListener('contextmenu', preventContextMenu);

    // Detectar Print Screen
    const preventPrintScreen = (e: KeyboardEvent) => {
      if (e.key === 'PrintScreen') {
        e.preventDefault();
        showWarning('⚠️ Las capturas de pantalla están deshabilitadas');
      }
    };
    document.addEventListener('keydown', preventPrintScreen);

    // Detectar herramientas de desarrollo
    const preventDevTools = () => {
      if (window.devtools?.open) {
        window.location.reload();
      }
    };

    return () => {
      document.removeEventListener('contextmenu', preventContextMenu);
      document.removeEventListener('keydown', preventPrintScreen);
    };
  }, [isUnlocked]);

  return <div className="protected-content">{children}</div>;
};
```

#### **Nivel 3: Watermarks Dinámicos**
```typescript
// lib/content-protection/watermark-generator.ts
export const generateWatermark = (userId: string, productId: string) => {
  const timestamp = new Date().toISOString();
  const userData = `Usuario: ${userId} | Producto: ${productId} | ${timestamp}`;
  
  return {
    text: userData,
    opacity: 0.15,
    position: 'center',
    rotation: -45,
    fontSize: '14px',
    color: '#000000'
  };
};

// Componente de watermark
export const WatermarkOverlay = ({ userId, productId }: Props) => (
  <div className="absolute inset-0 pointer-events-none z-20">
    <svg className="w-full h-full">
      <defs>
        <pattern id="watermark" x="0" y="0" width="300" height="200" patternUnits="userSpaceOnUse">
          <text 
            x="150" 
            y="100" 
            textAnchor="middle" 
            fill="black" 
            opacity="0.15"
            transform="rotate(-45 150 100)"
            fontSize="12"
          >
            {generateWatermark(userId, productId).text}
          </text>
        </pattern>
      </defs>
      <rect width="100%" height="100%" fill="url(#watermark)" />
    </svg>
  </div>
);
```

### **🚫 Medidas Anti-Descarga**

#### **Protección de Imágenes**
```typescript
// lib/content-protection/image-protection.ts
export const ProtectedImage = ({ src, alt, isUnlocked }: Props) => {
  const [isDragging, setIsDragging] = useState(false);

  const preventDrag = (e: React.DragEvent) => {
    e.preventDefault();
    setIsDragging(false);
  };

  const preventSave = (e: React.MouseEvent) => {
    e.preventDefault();
    if (!isUnlocked) {
      showWarning('🔒 Compra el producto para acceder al contenido completo');
    }
  };

  return (
    <div className="relative">
      <img
        src={isUnlocked ? src : `${src}?blur=12`}
        alt={alt}
        className={`${!isUnlocked ? 'blur-xl' : ''} select-none`}
        draggable={false}
        onDragStart={preventDrag}
        onMouseDown={preventSave}
        onContextMenu={preventSave}
      />
      {!isUnlocked && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/20">
          <div className="text-white text-center p-4">
            <Lock className="w-8 h-8 mx-auto mb-2" />
            <p>Contenido bloqueado</p>
            <p className="text-sm">Compra para desbloquear</p>
          </div>
        </div>
      )}
    </div>
  );
};
```

#### **Protección de Videos**
```typescript
// lib/content-protection/video-protection.ts
export const ProtectedVideo = ({ videoUrl, isUnlocked }: Props) => {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (!videoRef.current || !isUnlocked) return;

    // Deshabilitar controles de descarga
    videoRef.current.disablePictureInPicture = true;
    
    // Prevenir descarga via right-click
    const preventVideoMenu = (e: MouseEvent) => e.preventDefault();
    videoRef.current.addEventListener('contextmenu', preventVideoMenu);

    return () => {
      videoRef.current?.removeEventListener('contextmenu', preventVideoMenu);
    };
  }, [isUnlocked]);

  return (
    <div className="relative">
      <video
        ref={videoRef}
        src={isUnlocked ? videoUrl : ''}
        controls={isUnlocked}
        controlsList="nodownload noplaybackrate"
        disablePictureInPicture
        className="w-full"
        onContextMenu={(e) => e.preventDefault()}
      />
      {!isUnlocked && (
        <div className="absolute inset-0 flex items-center justify-center bg-black/50">
          <div className="text-white text-center">
            <Play className="w-12 h-12 mx-auto mb-2 opacity-50" />
            <p>Video bloqueado</p>
          </div>
        </div>
      )}
    </div>
  );
};
```

### **🔐 URLs Firmadas y Expiración**

#### **Generación de URLs Temporales**
```typescript
// lib/content-protection/secure-urls.ts
export const generateSecureUrl = (userId: string, productId: string, expiresMinutes = 10) => {
  const expires = Date.now() + (expiresMinutes * 60 * 1000);
  const token = jwt.sign(
    { userId, productId, expires, type: 'content_access' },
    process.env.CONTENT_SECRET!,
    { expiresIn: `${expiresMinutes}m` }
  );
  
  return `/api/content/${productId}/access?token=${token}`;
};

// Middleware de validación
export const validateContentAccess = async (req: Request) => {
  const token = req.nextUrl.searchParams.get('token');
  
  if (!token) {
    return { valid: false, reason: 'Token requerido' };
  }

  try {
    const decoded = jwt.verify(token, process.env.CONTENT_SECRET!) as any;
    
    // Verificar expiración
    if (Date.now() > decoded.expires) {
      return { valid: false, reason: 'Token expirado' };
    }

    // Verificar acceso en BD
    const hasAccess = await checkUserContentAccess(decoded.userId, decoded.productId);
    
    return { 
      valid: hasAccess, 
      userId: decoded.userId, 
      productId: decoded.productId,
      reason: hasAccess ? null : 'Acceso no autorizado'
    };
  } catch (error) {
    return { valid: false, reason: 'Token inválido' };
  }
};
```

### **📊 Detección de Usos Sospechosos**

#### **Monitoreo de Actividad Avanzado**
```typescript
// lib/content-protection/activity-monitor.ts
export const ContentActivityMonitor = ({ userId, productId }: Props) => {
  const [suspiciousActivity, setSuspiciousActivity] = useState(0);

  useEffect(() => {
    // Detectar múltiples intentos de captura
    let screenshotAttempts = 0;
    
    const detectScreenshot = () => {
      screenshotAttempts++;
      setSuspiciousActivity(screenshotAttempts);
      
      if (screenshotAttempts >= 3) {
        // Notificar al administrador
        notifySuspiciousActivity(userId, productId, 'multiple_screenshots');
        // Temporalmente bloquear acceso
        lockContentTemporarily(userId, productId, 5); // 5 minutos
      }
    };

    // Listener para detectar screenshots (limitado pero útil)
    document.addEventListener('visibilitychange', () => {
      if (document.hidden) {
        detectScreenshot();
      }
    });

    // Rate limiting por IP
    const trackAccessByIP = async () => {
      const clientIP = await getClientIP();
      const accessCount = await getAccessCountByIP(clientIP, productId);
      
      if (accessCount > 10) { // Más de 10 accesos en 1 hora
        notifySuspiciousActivity(userId, productId, 'excessive_ip_access');
        blockIPAccess(clientIP, 30); // Bloquear 30 minutos
      }
    };

    // Device fingerprinting
    const deviceFingerprint = generateDeviceFingerprint();
    await logDeviceAccess(userId, productId, deviceFingerprint);

    // Geolocalización y acceso simultáneo
    const checkMultipleDevices = async () => {
      const activeDevices = await getActiveUserDevices(userId);
      
      if (activeDevices.length > 2) {
        notifySuspiciousActivity(userId, productId, 'multiple_devices');
        // Opcional: permitir solo 2 dispositivos simultáneos
      }
    };

    trackAccessByIP();
    checkMultipleDevices();

    return () => {
      document.removeEventListener('visibilitychange', detectScreenshot);
    };
  }, [userId, productId]);

  return null;
};
```

#### **Sistema de Revocación de Acceso**
```typescript
// lib/content-protection/access-revocation.ts
export const revokeUserAccess = async (userId: string, productId: string, reason: string) => {
  // Marcar acceso como revocado en BD
  await db.update(user_unlocked_content)
    .set({ 
      status: 'revoked', 
      revokedAt: new Date(),
      revokedReason: reason
    })
    .where(
      and(
        eq(user_unlocked_content.userId, userId),
        eq(user_unlocked_content.productId, productId)
      )
    );

  // Invalidar todos los tokens activos
  await invalidateUserTokens(userId, productId);

  // Notificar al usuario
  await sendRevocationNotification(userId, productId, reason);

  // Log de auditoría
  await logAccessRevocation(userId, productId, reason);
};

// Endpoint para revocación admin
export async function POST(req: Request) {
  const { userId, productId, reason } = await req.json();
  
  // Verificar permisos de admin
  const session = await getServerSession();
  if (!session?.user?.role === 'admin') {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  await revokeUserAccess(userId, productId, reason);
  
  return NextResponse.json({ success: true });
}
```

#### **Hash de Archivos para Detección de Redistribución**
```typescript
// lib/content-protection/content-hashing.ts
export const generateContentHash = async (contentUrl: string) => {
  const response = await fetch(contentUrl);
  const buffer = await response.arrayBuffer();
  const hash = await crypto.subtle.digest('SHA-256', buffer);
  return Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
};

// Almacenar hash original al subir contenido
export const storeContentHash = async (productId: string, contentUrl: string) => {
  const hash = await generateContentHash(contentUrl);
  await db.insert(content_hashes).values({
    productId,
    originalUrl: contentUrl,
    hash,
    createdAt: new Date()
  });
};

// Servicio para detectar redistribución (webhook externo)
export const detectRedistribution = async (suspectedUrl: string) => {
  const suspectedHash = await generateContentHash(suspectedUrl);
  
  const match = await db.select()
    .from(content_hashes)
    .where(eq(content_hashes.hash, suspectedHash));
  
  if (match.length > 0) {
    // Contenido redistribuido detectado
    await notifyRedistribution(match[0].productId, suspectedUrl);
    return { detected: true, productId: match[0].productId };
  }
  
  return { detected: false };
};
```

#### **Videos con HLS Encriptado (Opcional Premium)**
```typescript
// lib/content-protection/hls-encryption.ts
export const generateHLSStream = async (videoPath: string, productId: string) => {
  // Usar FFmpeg para convertir a HLS con encriptación
  const command = `
    ffmpeg -i ${videoPath} \
    -hls_time 10 \
    -hls_key_info_file ${generateKeyFile(productId)} \
    -hls_playlist_type vod \
    ${videoPath}.m3u8
  `;
  
  await executeCommand(command);
  
  return {
    playlistUrl: `${videoPath}.m3u8`,
    keyUrl: `/api/content/${productId}/key`,
    encrypted: true
  };
};

// Endpoint para servir clave de desencriptación
export async function GET(req: Request, { params }: { params: { productId: string } }) {
  const token = req.nextUrl.searchParams.get('token');
  const access = await validateContentAccess(req);
  
  if (!access.valid) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }
  
  // Servir clave solo si tiene acceso válido
  const key = await getDecryptionKey(params.productId);
  
  return new NextResponse(key, {
    headers: {
      'Content-Type': 'application/octet-stream',
      'Cache-Control': 'no-cache, no-store, must-revalidate'
    }
  });
}
```

### **⚖️ Marco Legal y Términos**

#### **Términos de Uso Obligatorios**
```typescript
// components/content-protection/UsageTerms.tsx
export const UsageTermsModal = ({ onAccept }: Props) => (
  <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
    <div className="bg-white rounded-lg p-6 max-w-md mx-4">
      <h3 className="text-lg font-bold mb-4">📋 Términos de Uso del Contenido</h3>
      
      <div className="space-y-3 text-sm">
        <p>🚫 <strong>Prohibido:</strong></p>
        <ul className="list-disc list-inside space-y-1 text-gray-600">
          <li>Capturas de pantalla o grabaciones</li>
          <li>Compartir el contenido con terceros</li>
          <li>Descargar o redistribuir material</li>
          <li>Usar para fines comerciales no autorizados</li>
        </ul>
        
        <p>✅ <strong>Permitido:</strong></p>
        <ul className="list-disc list-inside space-y-1 text-gray-600">
          <li>Visualización personal del contenido</li>
          <li>Acceso desde "Mis Compras" únicamente</li>
          <li>Uso para aprendizaje individual</li>
        </ul>
      </div>
      
      <div className="mt-6">
        <label className="flex items-center">
          <input type="checkbox" className="mr-2" />
          <span className="text-sm">Acepto los términos y condiciones</span>
        </label>
      </div>
      
      <button 
        onClick={onAccept}
        className="w-full mt-4 bg-blue-600 text-white py-2 rounded hover:bg-blue-700"
      >
        Aceptar y Continuar
      </button>
    </div>
  </div>
);
```

---

## 🛠️ **Panel Administrativo - Validación de Pagos**

### **Dashboard Principal**
```
📊 Panel de Validación de Pagos
├── 🟡 Pagos Pendientes: 23
├── ✅ Pagos Aprobados Hoy: 45
├── ❌ Pagos Rechazados Hoy: 3
└── 💰 Total Procesado: $125,430
```

### **Gestión de Comprobantes**
```
📋 Lista de Órdenes por Validar
├── 🔍 Filtros: Fecha | Usuario | Monto | Estado
├── 📄 Vista Previa del Comprobante
├── ✅ Aprobar (con notas opcionales)
├── ❌ Rechazar (motivo obligatorio)
└── 📝 Historial de validaciones
```

### **Flujo de Validación**
```
1. Orden nueva aparece en "Pendientes"
2. Admin hace clic → Vista detallada:
   - Datos del usuario
   - Productos comprados
   - Imagen del comprobante (zoom)
   - Texto descriptivo del pago
3. Admin decide:
   ✅ Aprobar → Desbloquear contenido automáticamente
   ❌ Rechazar → Enviar notificación al usuario
   📝 Solicitar más info → Mantener en pendiente
```

### **Notificaciones Automáticas**
```
✅ Pago Aprobado:
   - Email al usuario con acceso
   - Notificación en panel
   - Registro en auditoría

❌ Pago Rechazado:
   - Email explicando motivo
   - Opción de subir nuevo comprobante
   - Marcar orden como "payment_rejected"
```

---

## 📁 **Estructura del Proyecto**

### **Carpetas Principales**
```
contenido-pagos-manuales/
├── app/
│   ├── (auth)/                    # Rutas de autenticación
│   ├── (protected)/               # Rutas protegidas
│   │   ├── products/              # Catálogo y vista de productos
│   │   ├── cart/                  # Carrito de compras
│   │   ├── checkout/              # Checkout manual
│   │   ├── orders/                # "Mis compras" del usuario
│   │   └── profile/               # Perfil y contenido desbloqueado
│   ├── admin/                     # Panel administrativo
│   │   ├── dashboard/             # Métricas generales
│   │   ├── payments/              # Validación de comprobantes
│   │   ├── products/              # Gestión de productos
│   │   └── users/                 # Gestión de usuarios
│   ├── api/                       # API Routes
│   │   ├── auth/                  # NextAuth endpoints
│   │   ├── products/              # CRUD productos
│   │   ├── cart/                  # Gestión carrito
│   │   ├── orders/                # Órdenes y comprobantes
│   │   └── admin/                 # Endpoints admin
│   └── globals.css, layout.tsx    # Configuración base
├── components/
│   ├── ui/                        # Componentes base (reutilizados)
│   ├── products/                  # Cards, galería, vista detalle
│   ├── cart/                      # Carrito y resumen
│   ├── checkout/                  # Formulario de comprobante
│   ├── admin/                     # Panel de validación
│   └── layout/                    # Header, footer, navegación
├── lib/
│   ├── auth/                      # Configuración NextAuth
│   ├── db.ts                      # Conexión a base de datos
│   ├── schema.ts                  # Esquemas Drizzle
│   ├── actions/                   # Server actions
│   │   ├── products.ts            # CRUD productos
│   │   ├── cart.ts                # Gestión carrito
│   │   ├── orders.ts              # Órdenes y pagos
│   │   └── admin.ts               # Acciones admin
│   ├── stores/                    # Zustand stores
│   │   └── cart-store.ts          # Estado del carrito
│   ├── utils/                     # Utilidades generales
│   └── validations/               # Zod schemas
├── public/                        # Assets estáticos
├── drizzle/                       # Migraciones de BD
└── types/                         # Tipos TypeScript
```

---

## 🔧 **Componentes Clave a Desarrollar**

### **1. Galería Multimedia de Productos**
```tsx
// components/products/ProductMediaGallery.tsx
interface ProductMediaGalleryProps {
  mainImage: string;
  secondaryImages: string[];
  videos: VideoItem[];
  isUnlocked: boolean; // ¿Contenido desbloqueado?
}

// Features:
- Navegación entre imágenes
- Video player con thumbnail
- Indicador de contenido bloqueado
- Zoom en imágenes
- Lazy loading
```

### **2. Formulario de Comprobante**
```tsx
// components/checkout/PaymentProofForm.tsx
interface PaymentProofFormProps {
  orderId: string;
  onsubmit: (data: PaymentProofData) => void;
}

// Features:
- Upload de imagen con validación
- Campo de texto para detalles
- Preview del comprobante
- Validación de tamaño/formato
- Indicadores de progreso
```

### **3. Panel de Validación Admin**
```tsx
// components/admin/PaymentValidationPanel.tsx
interface PaymentValidationPanelProps {
  orders: OrderWithProof[];
  onApprove: (orderId: string) => void;
  onReject: (orderId: string, reason: string) => void;
}

// Features:
- Vista en cuadrícula/lista
- Filtros avanzados
- Preview de comprobantes
- Acciones en lote
- Historial de cambios
```

---

## 📊 **Base de Datos - Schema Detallado**

### **Schema Products (Modificado)**
```typescript
export const products = pgTable('products', {
  id: serial('id').primaryKey(),
  title: text('title').notNull(),
  description: text('description').notNull(),
  price: decimal('price', { precision: 10, scale: 2 }).notNull(),
  salePrice: decimal('sale_price', { precision: 10, scale: 2 }),
  categoryId: integer('category_id').references(() => categories.id),
  status: text('status').default('active'), // active/inactive
  
  // Campos multimedia
  mainImageUrl: text('main_image_url'),
  secondaryImage1: text('secondary_image_1'),
  secondaryImage2: text('secondary_image_2'),
  secondaryImage3: text('secondary_image_3'),
  videoUrl: text('video_url'),
  videoThumbnail: text('video_thumbnail'),
  
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});
```

### **Schema Orders (Simplificado)**
```typescript
export const orders = pgTable('orders', {
  id: serial('id').primaryKey(),
  userId: integer('user_id').references(() => users.id),
  totalAmount: decimal('total_amount', { precision: 10, scale: 2 }).notNull(),
  status: text('status').default('pending'), // pending/completed/cancelled
  paymentStatus: text('payment_status').default('payment_pending'), // payment_pending/approved/rejected
  adminNotes: text('admin_notes'),
  proofImageUrl: text('proof_image_url'),
  proofText: text('proof_text'),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});
```

### **Schema Payment Proofs (Nuevo)**
```typescript
export const paymentProofs = pgTable('payment_proofs', {
  id: serial('id').primaryKey(),
  orderId: integer('order_id').references(() => orders.id),
  userId: integer('user_id').references(() => users.id),
  imageUrl: text('image_url').notNull(),
  proofText: text('proof_text'),
  status: text('status').default('pending'), // pending/approved/rejected
  adminNotes: text('admin_notes'),
  validatedBy: integer('validated_by').references(() => users.id),
  validatedAt: timestamp('validated_at'),
  createdAt: timestamp('created_at').defaultNow(),
});
```

---

## 🔄 **Flujos de API Principales**

### **1. Creación de Orden con Comprobante**
```typescript
// POST /api/orders
{
  "items": [{ "productId": 1, "quantity": 1 }],
  "paymentProof": {
    "imageUrl": "https://cdn.com/proof.jpg",
    "proofText": "Transferencia Bancaria - $50.00 - Referencia 12345"
  }
}

// Response:
{
  "orderId": 123,
  "status": "payment_pending",
  "message": "Orden creada. Esperando validación del comprobante."
}
```

### **2. Validación Admin**
```typescript
// POST /api/admin/payments/:orderId/approve
{
  "adminNotes": "Comprobante validado correctamente"
}

// POST /api/admin/payments/:orderId/reject
{
  "reason": "El monto no coincide con el total de la orden",
  "adminNotes": "Se solicitó al usuario subir nuevo comprobante"
}
```

### **3. Verificación de Contenido Desbloqueado**
```typescript
// GET /api/products/:productId/access?userId=X
{
  "hasAccess": true,
  "unlockedAt": "2024-01-15T10:30:00Z",
  "expiresAt": null,
  "content": {
    "fullDescription": "...",
    "videos": ["..."],
    "downloadableFiles": ["..."]
  }
}
```

---

## 🔒 **Seguridad y Validaciones**

### **Validaciones de Comprobantes**
- **Formato de imagen**: Solo JPG/PNG/WebP
- **Tamaño máximo**: 5MB por imagen
- **Texto obligatorio**: Mínimo 10 caracteres
- **Detección de duplicados**: Hash de imagen
- **Rate limiting**: 3 comprobantes por hora por usuario

### **Permisos de Administrador**
- **Roles**: admin, moderator, viewer
- **Acciones por rol**:
  - Admin: Aprobar/rechazar + eliminar productos
  - Moderator: Aprobar/rechazar + editar productos
  - Viewer: Solo ver panel de métricas

### **Seguridad de Contenido**
- **Acceso por usuario**: Verificación JWT + BD
- **Watermark en imágenes**: Protección de contenido
- **URLs temporales**: Links firmados para contenido
- **Auditoría completa**: Log de todas las acciones

---

## 📱 **Experiencia de Usuario**

### **Notificaciones y Feedback**
```
✅ Estados claros en cada paso:
   - "Carrito actualizado"
   - "Comprobante subido correctamente"
   - "Pago en revisión (24-48hs)"
   - "¡Pago aprobado! Contenido desbloqueado"

🔔 Notificaciones automáticas:
   - Email de confirmación de orden
   - Email de aprobación/rechazo
   - Notificaciones en panel (real-time)
```

### **Diseño Responsivo**
```
📱 Mobile-first:
   - Galería táctil de imágenes
   - Upload fácil de comprobantes
   - Panel admin optimizado móvil

💻 Desktop:
   - Vista previa de comprobantes grande
   - Validación en lote
   - Atajos de teclado
```

---

## 🚀 **Plan de Implementación - Fases**

### **FASE 1: Base del Proyecto y Protección (2 semanas)**
```
✅ Setup inicial del proyecto
✅ Base de datos y migraciones
✅ Autenticación básica
✅ CRUD de productos multimedia
✅ Carrito de compras funcional
✅ Sistema de BLUR para contenido bloqueado
✅ Protección básica anti-captura
```

### **FASE 2: Sistema de Pagos (2 semanas)**
```
✅ Formulario de comprobantes
✅ Creación de órdenes con proof
✅ Panel admin básico
✅ Sistema de notificaciones
✅ Validación manual de pagos
```

### **FASE 3: Desbloqueo de Contenido (1 semana)**
```
✅ Sistema de acceso por usuario
✅ Verificación de contenido desbloqueado
✅ Panel de usuario con compras
✅ Historial de accesos
```

### **FASE 4: Mejoras y Polish (1 semana)**
```
✅ Testing completo
✅ Optimización de imágenes
✅ Mejoras UX/UI
✅ Documentación
✅ Deploy a producción
```

---

## 📊 **Métricas de Éxito**

### **KPIs Principales**
- **Tasa de conversión**: % usuarios que completan compra
- **Tiempo de validación**: Promedio admin para aprobar pagos
- **Satisfacción usuario**: Feedback post-compra
- **Contenido desbloqueado**: Accesos únicos vs compras

### **Métricas Operativas**
- **Comprobantes por día**: Volumen de validaciones
- **Rechazos por motivo**: Identificar problemas comunes
- **Tiempo en cola**: Tiempo espera usuario promedio
- **Productividad admin**: Órdenes procesadas por hora

---

## 🛠️ **Herramientas y Servicios**

### **Infraestructura Recomendada**
```
🌐 Hosting: Vercel (Next.js optimizado)
🗄️ Base de datos: Neon (PostgreSQL serverless)
📧 Email: Resend (notificaciones transaccionales)
🖼️ Image CDN: Cloudinary/Vercel Blob
📊 Analytics: Vercel Analytics + Google Analytics
```

### **Herramientas de Desarrollo**
```
🔧 Stack: Next.js 15 + TypeScript + Tailwind
🗃️ ORM: Drizzle (mismas ventajas que proyecto actual)
🎨 UI: shadcn/ui + componentes personalizados
🧪 Testing: Jest + Testing Library
📝 Docs: Storybook para componentes
```

---

## 💰 **Modelo de Negocio y Monetización**

### **Para el Administrador**
```
💳 Procesamiento manual = Control total
📊 Métricas en tiempo real
🎯 Segmentación de usuarios
📈 Escalabilidad sin comisiones externas
```

### **Para los Usuarios**
```
🔒 Contenido exclusivo y seguro
💎 Acceso permanente una vez comprado
📱 Experiencia simple y directa
🛡️ Comprobantes de pago privados
```

---

## 🎯 **Próximos Pasos**

### **Inmediato (Esta semana)**
1. **Fork del proyecto actual** - Eliminar módulos ML/MP
2. **Definir nuevo schema** - Crear migraciones base
3. **Setup autenticación simplificada** - Sin OAuth ML
4. **Diseñar componente galería** - Imágenes + videos

### **Corto Plazo (2 semanas)**
1. **Implementar carrito básico** - Reutilizar lógica existente
2. **Crear formulario comprobantes** - Upload + validación
3. **Desarrollar panel admin** - Vista de validación
4. **Sistema de notificaciones** - Email básico

### **Mediano Plazo (1 mes)**
1. **Testing completo** - Flujo end-to-end
2. **Optimización performance** - Imágenes y CDN
3. **Documentación API** - Endpoints públicos
4. **Deploy producción** - Configuración final

---

## 📝 **Conclusión**

Este proyecto aprovecha la sólida base arquitectónica del e-commerce actual pero la adapta a un modelo de negocio más simple y controlado. La eliminación de integraciones externas complejas (Mercado Libre/Pago) reduce significativamente la superficie de errores y mantenimiento, mientras que el sistema de validación manual ofrece control total sobre el proceso de pago.

**Ventajas principales vs proyecto actual:**
- ✅ **Mantenimiento simplificado** - Sin dependencias externas
- ✅ **Control total** - Validación manual de cada transacción  
- ✅ **Costos reducidos** - Sin comisiones de pasarelas
- ✅ **Contenido protegido** - Acceso granular por usuario
- ✅ **Escalabilidad predecible** - Crecimiento lineal sin APIs externas

El tiempo estimado de desarrollo es de **6 semanas** para un MVP funcional, con potencial de expansión futura basado en la demanda del mercado.

---

**Estado del Plan:** ✅ **Completo y listo para ejecución**
**Próxima acción:** Fork del proyecto actual y eliminación de módulos ML/MP
