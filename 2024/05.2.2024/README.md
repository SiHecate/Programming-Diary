# Özet

Online ödeme sistemi için gereken verilerin toplanması hakkında çalışmalarda bulundum.
Bu çalışmalar içerisinde UserInfo adında bir repository ve service oluşturdum ardından bunları controller'a bağladım. Burada aldığım bilgiler kullanıcının şu verilerini içermektedir:

```
    Adres adı,
    User Id,
    İsim,
    Soyisim,
    Email,
    Telefon,
    İl,
    İlçe,
    Mahalle,
    Tam adres
```

Şeklinde burada kullandığım dataları farklı online ödeme platformlarına bağlamak amacıyla kullanacağım.
Şu an için projeye eklemeyi düşündüğüm online ödeme sistemleri _iyzico_ ve _paytr_ iki online ödeme sisteminde de ortak veriler istendiği için sadece tek bir ödeme sistemine göre programı tasarlamaktansa her ödeme sistemini kapsayacak bir veri havuzu elde etmeye çalışacağım. Bu konuda araştırmalarım devam ediyor.

Kullanıcının bilgilerini aldığım service yapısı aşağıdadır:

Repository Interface

```
interface UserInfoRepositoryInterface
{
    public function getAll();

    public function getUserInfos($userId);

    public function create(array $data);

    public function update(array $data, $userId);

    public function delete($address_name, $userId);
}

```

Repository

```
class UserInfoRepository implements Interfaces\UserInfoRepositoryInterface
{
    public function getAll()
    {
        return UserInfo::orderBy('created_at', 'asc')->get();
    }

    public function getUserInfos($userId)
    {
        return UserInfo::where('user_id', $userId)->get();
    }

    public function create($data)
    {
        return UserInfo::create($data);
    }

    public function update($data, $userId)
    {
        return UserInfo::where('user_id', $userId)->update($data);
    }

    public function delete($userId, $addressName)
    {
        return UserInfo::where('user_id', $userId)->where('address_name', $addressName)->delete();
    }

}
```

Service

```
class UserInfoService
{
    protected $userInfoRepository;

    public function __construct(UserInfoRepositoryInterface $userInfoRepository)
    {
        $this->userInfoRepository = $userInfoRepository;
    }

    public function userInfos()
    {
        return $this->userInfoRepository->getAll();
    }

    public function getUserInfos($userId): JsonResponse
    {
        $userInfos = $this->userInfoRepository->getUserInfos($userId);

        if ($userInfos->isNotEmpty()) {
            $formattedUserInfos = $userInfos->map(function ($info) {
                return [
                    'address_name' => $info->address_name,
                    'name' => $info->name,
                    'surname' => $info->surname,
                    'email' => $info->email,
                    'telephone' => $info->telephone,
                    'city' => $info->city,
                    'district' => $info->district,
                    'neighborhood' => $info->neighborhood,
                    'address' => $info->address
                ];
            });
            return response()->json([
                'message' => 'user_infos',
                'data' => $formattedUserInfos,
            ]);
        }
        return response()->json([
            'message' => 'user_info not found',
            'data' => [],
        ]);
    }


    public function createUserInfo(array $data, $userId): JsonResponse
    {
        $createdUserInfo = $this->userInfoRepository->create($data, $userId);

        if ($createdUserInfo) {
            return response()->json(['message' => 'User info created successfully', 'data' => $createdUserInfo], 201);
        } else {
            return response()->json(['message' => 'Failed to create user info'], 500);
        }
    }

    public function updateUserInfo(array $data, $userId): JsonResponse
    {
        $updatedUserInfo = $this->userInfoRepository->update($data, $userId);

        if ($updatedUserInfo) {
            return response()->json(['message' => 'User info updated successfully', 'userInfo' => $updatedUserInfo], 200);
        } else {
            return response()->json(['message' => 'Failed to update user info'], 500);
        }
    }

    public function deleteUserInfo($addressName, $userId): JsonResponse
    {
        $deletedUserInfo = $this->userInfoRepository->delete($userId, $addressName);

        if ($deletedUserInfo) {
            return response()->json(['message' => 'User info deleted successfully'], 200);
        } else {
            return response()->json(['message' => 'Failed to delete user info'], 500);
        }
    }
 }

```

Controller

```
class UserInfoController extends Controller
{
    protected $userInfoRepository;
    protected $userInfoService;

    public function __construct(UserInfoRepository $userInfoRepository, UserInfoService $userInfoService)
    {
        $this->userInfoRepository = $userInfoRepository;
        $this->userInfoService = $userInfoService;
    }

    public function index()
    {
        $allUserInfo = $this->userInfoService->userInfos();
        return $allUserInfo;
    }

    public function store(UserInfoRequest $request)
    {
        $userId = $request->user()->id;
        $validatedData = $request->validate();
        return $this->userInfoService->createUserInfo($validatedData, $userId);
    }

    public function update(UserInfoRequest $request)
    {
        $userId = $request->user()->id;
        $validatedData = $request->validated();
        return $this->userInfoService->updateUserInfo($validatedData, $userId);
    }

    public function delete(UserInfoRequest $request)
    {
        $userId = $request->user()->id;
        $validatedData = $request->validated();
        $addressName = $validatedData['address_name'];
        return $this->userInfoService->deleteUserInfo($addressName, $userId);
    }

    public function show(UserInfoRequest $request)
    {
        $userId = $request->user()->id;
        return $this->userInfoService->getUserInfos($userId);
    }

}
```

Bu fonksiyonlar içerisinde güncellemeler gerekecek fakat bu ödeme sistemini kararlaştırdıktan sonraya bıraktım.

Yarından itibaren online ödeme sistemlerinin entegrasyonu hakkında bilgi edinmeye başlayacağım.
