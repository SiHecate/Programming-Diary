# 2/20/2024 Özeti

Alışveriş sitesi uygulamamda ödeme sistemleriyle ilgili çalışmalar yapıyorum. Ödeme sistemlerini incelediğimde, genellikle istenen verilerin neredeyse aynı olduğunu fark ettim. Bu nedenle, farklı ödeme sistemlerini projeme entegre ederken, her bir ödeme sistemi için kullanıcıdan veri almaktansa bu işlemi bir hizmet (service) katmanında yönetmeyi tercih ediyorum. Böylece, farklı ödeme sistemlerini entegre ve test ederken kullanıcıdan tekrar tekrar aynı verileri istemek zorunda kalmam.

## Ödeme sistemi

Burada PayTR api entegrasyon'u yapıyorum benden istediği verileri service katmanları aracılığı ile dolduruyorum.

Referanslar:

- https://github.com/serkanikizoglu/Laravel-PayTr-Odeme-Entegrasyonu

- https://github.com/GizemSever/laravel-paytr

```
class PaymentService
{

    protected $userInfoService;
    protected $basketService;

    public function __construct(UserInfoService $userInfoService, BasketService $basketService)
    {
        $this->userInfoService->$userInfoService;
        $this->basketService->$basketService;
    }

    public function index()
    {

    }

    public function randomNumberGenerator()
    {
        $randomNumber = "11".rand(1,999).rand(1,88)*rand(1,50);
        return $randomNumber;
    }

    public function payment()
    {

        $data =request()->all();
        $userId = auth()->id();
        $userInfos = $this->userInfoService->getUserInfos($userId);
        $basketInfos = $this->basketService->getBasket($userId);

        $merchant_id    = 'xxxxxxx';
        $merchant_key   = 'xxxxxxx';
        $merchant_salt  = 'xxxxxxx';
        $email = auth()->user()->email;
        $payment_amount = ($basketInfos->total_price)*100; //9.99 için 9.99 * 100 = 999 gönderilmelidir.
        $merchant_oid = $this->randomNumberGenerator();
        $user_name = $userInfos->name;
        $user_address = $userInfos->address;
        $user_phone = $userInfos->telephone;
        $merchant_ok_url = redirect()->route('success');
        $merchant_fail_url = redirect()->route('error');
        $user_basket = base64_encode(json_encode(array(
            array($data["urun"], $payment_amount, 1)
        )));

        #
        /* ÖRNEK $user_basket oluşturma - Ürün adedine göre array'leri çoğaltabilirsiniz
        $user_basket = base64_encode(json_encode(array(
            array("Örnek ürün 1", "18.00", 1), // 1. ürün (Ürün Ad - Birim Fiyat - Adet )
            array("Örnek ürün 2", "33.25", 2), // 2. ürün (Ürün Ad - Birim Fiyat - Adet )
            array("Örnek ürün 3", "45.42", 1)  // 3. ürün (Ürün Ad - Birim Fiyat - Adet )
        )));
        */
        ############################################################################################

        ## Kullanıcının IP adresi
        if( isset( $_SERVER["HTTP_CLIENT_IP"] ) ) {
            $ip = $_SERVER["HTTP_CLIENT_IP"];
        } elseif( isset( $_SERVER["HTTP_X_FORWARDED_FOR"] ) ) {
            $ip = $_SERVER["HTTP_X_FORWARDED_FOR"];
        } else {
            $ip = $_SERVER["REMOTE_ADDR"];
        }

        $user_ip=$ip;

        $timeout_limit = "5";

        $debug_on = 1;

        $test_mode = 0;

        $no_installment = 1;

        $max_installment = 0;

        $currency = "TL";

        $hash_str = $merchant_id .$user_ip .$merchant_oid .$email .$payment_amount .$user_basket.$no_installment.$max_installment.$currency.$test_mode;
        $paytr_token=base64_encode(hash_hmac('sha256',$hash_str.$merchant_salt,$merchant_key,true));
        $data_vals=array(
            'merchant_id'=>$merchant_id,
            'user_ip'=>$user_ip,
            'merchant_oid'=>$merchant_oid,
            'email'=>$email,
            'payment_amount'=>$payment_amount,
            'paytr_token'=>$paytr_token,
            'user_basket'=>$user_basket,
            'debug_on'=>$debug_on,
            'no_installment'=>$no_installment,
            'max_installment'=>$max_installment,
            'user_name'=>$user_name,
            'user_address'=>$user_address,
            'user_phone'=>$user_phone,
            'merchant_ok_url'=>$merchant_ok_url,
            'merchant_fail_url'=>$merchant_fail_url,
            'timeout_limit'=>$timeout_limit,
            'currency'=>$currency,
            'test_mode'=>$test_mode
        );

        $ch=curl_init();
        curl_setopt($ch, CURLOPT_URL, "https://www.paytr.com/odeme/api/get-token");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        curl_setopt($ch, CURLOPT_POST, 1) ;
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data_vals);
        curl_setopt($ch, CURLOPT_FRESH_CONNECT, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 20);

        $result = @curl_exec($ch);

        if(curl_errno($ch))
            die("PAYTR IFRAME connection error. err:".curl_error($ch));

        curl_close($ch);

        $result=json_decode($result,1);

        if($result['status']=='success')
            $token=$result['token'];
        else
            die("PAYTR IFRAME failed. reason:".$result['reason']);

        return view("odeme.sonuc",compact("token"));

    }

    public function notification(){
        $data = request()->all();

        $merchant_key   = 'xxxxxxx';
        $merchant_salt  = 'xxxxxxx';

        $hash = base64_encode( hash_hmac('sha256', $data['merchant_oid'].$merchant_salt.$data['status'].$data['total_amount'], $merchant_key, true) );

        if( $hash != $data['hash'] )
            die('PAYTR notification failed: bad hash');


        if( $data['status'] == 'success' ) { ## Ödeme Onaylandı


        } else { ## Ödemeye Onay Verilmedi


        }

        ## Bildirimin alındığını PayTR sistemine bildir.
        echo "OK";
        exit;

    }
}
```

## Purchase

Buradaki methodlar ise ödeme sistemi içerisinde kullanılacak olan fonksiyonlar

- Ürün satışı başarılı olursa ürünün stok sayısı kaç tane satıldıysa o kadar azaltılacak.
- Ürün satışı başarılı olursa kullanıcıya ürün bilgileri ve fatura adresi gibi bilgiler verilecek
- Ürün satışı başarısız olursa **şu an için** null döndürülüyor bunun için farklı bir şey düşünülecek.

```
class PurchaseService
{
    private $productRepository;
    private $basketService;
    private $userInfoRepository;

    public function __construct
    (
        ProductRepositoryInterface $productRepository,
        BasketService $basketService,
        UserInfoRepository $userInfoRepository,
    )
    {
        $this->productRepository = $productRepository;
        $this->basketService = $basketService;
        $this->userInfoRepository = $userInfoRepository;
    }

    public function purchase($productId, $purchaseQuantity)
    {
        $product = $this->productRepository->findProductById($productId);

        if (!$product) {
            throw new \Exception("Product not found.");
        }

        if ($product->quantity < $purchaseQuantity) {
            throw new \Exception("Not enough products in stock..");
        }

        $newQuantity = $product->quantity - $purchaseQuantity;

        $this->productRepository->update(['quantity' => $newQuantity], $productId);
    }

    public function success($userId)
    {
        /*
            Ürünlerin listesi olacak
                - Kaç tane alındığı
                - Toplam ücreti
                - Saat kaçta alındığı
                - Hangi adrese alındığı
                    'address_name' => $info->address_name,
                    'name' => $info->name,
                    'surname' => $info->surname,
                    'email' => $info->email,
                    'telephone' => $info->telephone,
                    'city' => $info->city,
                    'district' => $info->district,
                    'neighborhood' => $info->neighborhood,
                    'address' => $info->address

                                Buradaki bilgilere giriş yapan kullanıcının id'sinden çektirerek erişilebilir.

            Kullanıcının basket'i boşaltılacak
        */

        $basketData = $this->basketService->getBasket($userId);

        if ($basketData && isset($basketData['product_datas'])) {
            $products = $basketData['product_datas'];
            $totalPrice = $basketData['basket_total_price'];

            $userInfo = $this->userInfoRepository->getUserInfos($userId);

            if ($userInfo) {
                $userInfoArray = [
                    'name' => $userInfo->name,
                    'surname' => $userInfo->surname,
                    'telephone' => $userInfo->telephone,
                    'city' => $userInfo->city,
                    'district' => $userInfo->district,
                    'neighborhood' => $userInfo->neighborhood,
                    'full_address' => $userInfo->address,
                ];

                $response = [
                    'user_info' => $userInfoArray,
                    'products' => $products,
                    'total_price' => $totalPrice,
                    'purchase_time' => now(),
                ];

                $this->basketService->clearUserBasket($userId);

                return response()->json($response, 200);
            } else {
                return response()->json(['message' => 'User info cannot be found'], 404);
            }
        } else {
            return response()->json(['message' => 'Basket data cannot be found'], 404);
        }
    }


    public function failure(){
        return null;
    }
}
```

```
    public function getBasket($userId)
    {
        $basket = $this->basketRepository->findUserBasket($userId);
        $products = json_decode($basket->products, true) ?? [];

        $totalPrice = $this->calculateTotalPrice($products);;
        $productDetails = [];
        $productQuantity = [];
        foreach ($products as $product_id) {
            $product = $this->productService->findProductById($product_id);
            $productQuantity[$product_id] = isset($productQuantity[$product_id]) ? $productQuantity[$product_id] + 1 : 1;
            if ($product) {
                if (!isset($productDetails[$product->id])) {
                    $productDetails[$product->id] = [
                        'code' => $product->id,
                        'title' => $product->title,
                        'image' => $product->image,
                        'desc' => $product->description,
                        'price' => $product->price,
                        'quantity' => $productQuantity[$product_id],
                        'total_price' => $product->price * $productQuantity[$product_id],
                    ];
                } else {
                    $productDetails[$product->id]['quantity'] = $productQuantity[$product_id];
                    $productDetails[$product->id]['total_price'] = $product->price * $productQuantity[$product_id];
                }
            }
        }
        return response()->json(['product_datas' => array_values($productDetails),'basket' => $basket, 'basket_total_price' => $totalPrice], 200);
    }

```

```
    public function clearUserBasket($userId)
    {
        Basket::where('user_id', $userId)->update(['products' => '']);
    }
```

(Daha mantıklı bir yolunun olduğuna %100 eminim ileride tekrardan düzenleyeceğim)

## Route update

### Middleware

Kullanıcının admin olup olmadığının kontrolünü yapan Middleware

```
class AdminCheck
{
    /**
     * Handle an incoming request.
     *
     * @param  \Closure(\Illuminate\Http\Request): (\Symfony\Component\HttpFoundation\Response)  $next
     */
    public function handle(Request $request, Closure $next): Response
    {
        $userRole = Auth::user()->role;
        if ($userRole != 'Admin')
        {
            return response()->json([
                'error' => 'user not admin',
            ], 403);
        } else {
            return $next($request);
        }
    }
}
```

### Route

Middleware'lar kullanarak routing'i daha verimli hale getirdim

```
// Product routes
Route::prefix('product')->group(function () {

    Route::get('/', [ProductController::class, 'index'])->name('product.index');
    Route::post('/search', [ProductController::class, 'search'])->name('product.search');
    Route::get('/{id}', [ProductController::class, 'show'])->name('product.show');

    // Admin routes
    Route::middleware(['auth', AdminCheck::class])->group(function () {
        Route::post('/add', [ProductController::class, 'store'])->name('product.store');
        Route::put('/{id}', [ProductController::class, 'update'])->name('product.update');
        Route::delete('/{id}', [ProductController::class, 'destroy'])->name('product.destroy');
    });
});

// Basket routes
Route::prefix('basket')->middleware('auth')->group(function () {
    Route::get('/', [BasketController::class, 'index'])->name('basket.index');
    Route::post('/', [BasketController::class, 'store'])->name('basket.store');
    Route::put('/{id}/{type}', [BasketController::class, 'update'])->name('basket.update');
    Route::delete('/{id}', [BasketController::class, 'destroy'])->name('basket.destroy');
    Route::get('/sepet', [BasketController::class, 'view'])->name('basket.view');
    Route::get("/paymentBasket", [BasketController::class, 'basket'])->name('basket.payment');
});

// Address routes
Route::prefix('address')->middleware('auth')->group(function () {
    Route::get('/', [UserInfoController::class, 'index'])->name('address.index');
    Route::post('/', [UserInfoController::class, 'store'])->name('address.store');
    Route::put('/', [UserInfoController::class, 'update'])->name('address.update');
    Route::get('/info', [UserInfoController::class, 'show'])->name('address.show');
    Route::delete('/{id}', [UserInfoController::class, 'delete'])->name('address.delete');
});
```
