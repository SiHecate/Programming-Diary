# Özet

Ödeme yapıldığı eğer ödeme başarılı ise yönlendirilen fonksiyon içerisine düzenlemeler yapıldı.

- Ürün satışı başarılı olduğunda ürünün stoğu alınan ürün sayısına göre azaltılmasını sağlayan fonksiyon.
- Stok azaltıldığı zaman eğer stok tükenirse Product table'ı içerisindeki "görünürlük" observer design pattern yardımıyla 0'a yani false'a ürüne yeni stok eklendiği zamanda 1'e yani true'ya güncellenmesi sağlandı.
- Ürünlerin satışı başarılı olduğunda sepetteki bütün ürünlerin bir database table'ı içerisinde saklanması ve saklanan ürünlerin sonrasında kullanıcıya "Siparişlerim" adı altında gösterilmesini sağlayan service katmanı.

**Yapılacaklar**

- Webhook yardımıyla stripe servislerinden ürünün satışının yapıldığının bilgisinin alınması ve alınan bilgi ile Order database table'ında unpaid'i -> paid olarak güncellenmesi sağlanacak.
- Siparişlerim sayfası tam haline getirilecek.
- Uygulama içerisinde başka hangi yerlerde Observer design pattern kullanılabilir bunun araştırılması yapılacak.

## Yeni eklenen kodlar

### Order service

```
class OrderService
{

    // Construct

    private $productService;

    public function __construct(ProductService $productService)
    {
        $this->productService = $productService;
    }

    public function createOrder($userId, $totalPrice, $productId, $productQuantity)
    {
        $orderPanel = OrderPanel::create([
            'user_id' => $userId,
            'total_amount' => $totalPrice,
        ]);
        $product = $this->productService->findProductById($productId);
        $productDetail = [
            'order_id' => $orderPanel->id,
            'order_number' => mt_rand(100000, 999999),
            'product_name' => $product['title'],
            'product_image' => $product['image'],
            'product_price' => $product['price'],
            'product_quantity' => $productQuantity,
        ];
        OrderDetail::create($productDetail);
        return true;
    }
}
```

```
class OrderPanel extends Model
{
    use HasFactory;

    protected $fillable = ['order_id' ,'user_id', 'total_amount'];

    /**
     * Get the order details for the order panel.
     */
    public function orderDetails()
    {
        return $this->hasMany(OrderDetail::class);
    }
}
```

```
class OrderDetail extends Model
{
    use HasFactory;

    protected $fillable = ['order_id', 'order_number', 'product_name', 'product_image', 'product_price', 'product_quantity'];

    /**
     * Get the order panel that owns the order detail.
     */
    public function orderPanel()
    {
        return $this->belongsTo(OrderPanel::class);
    }
}

```

### Product observer

```
class ProductObserver
{
        public function updated(Product $product): void
    {
        if ($product->stock <= 0) {
            $product->visibility = 0;
        } elseif ($product->stock >= 1){
            $product->visibility = 1;
        }
    }
}
```

```
class Product extends Model
{
    use HasFactory;

    protected $fillable = ['title', 'description', 'image', 'price', 'stock', 'visibility', 'tag'];


    protected static function boot()
    {
        parent::boot();

        static::observe(ProductObserver::class);
    }
}
```

### Güncellenen success fonksiyonu (Ödeme sistemi içerisinde kullanılıyor)

```
    public function success($userId)
    {
        $basketData = json_decode($this->basketService->getBasket($userId)->getContent(), true);
        if ($basketData && isset($basketData['product_datas'])) {
            $products = $basketData['product_datas'];
            $totalPrice = $basketData['basket_total_price'];
            $userInfo = $this->userInfoService->getUserInfos($userId);
            if ($userInfo) {
                $response = [
                    'user_info' => $userInfo,
                    'products' => $products,
                    'total_price' => $totalPrice,
                    'purchase_time' => now(),
                ];
            foreach($products as $product)
            {
                $quantity = $product['quantity'];
                $productId = $product['code'];
                $this->productService->stockReduction($productId, $quantity);
                $this->orderService->createOrder($userId, $totalPrice, $productId, $quantity);
            }
                $this->basketService->clearUserBasket($userId);
                return response()->json($response, 200);
            } else {
                return response()->json(['message' => 'User info cannot be found'], 404);
            }
        } else {
            return response()->json(['message' => 'Basket data cannot be found'], 404);
        }
    }
```
