import SwiftUI
import MapKit
import CoreLocation

@main
struct MapApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// SwiftUIのContentView
struct ContentView: View {
    @StateObject private var mapViewModel = MapViewModel()
    
    var body: some View {
        ZStack {
            MapView(mapViewModel: mapViewModel)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                Spacer()  // 検索バーを一番下に配置するためのスペーサー
                
                // 検索バーの追加
                HStack(spacing: 0) {
                    TextField("場所を検索", text: $mapViewModel.searchText)
                        .textFieldStyle(RoundedBorderTextFieldStyle())
                        .padding(.horizontal, 10)
                        .frame(height: 50)
                    
                    Button(action: {
                        mapViewModel.searchLocation()
                    }) {
                        Text("検索")
                            .frame(maxWidth: .infinity, maxHeight: .infinity)
                            .padding()
                            .background(Color.blue)
                            .foregroundColor(.white)
                    }
                    .frame(width: 100, height: 50)
                    .cornerRadius(1)
                }
                .background(Color.white)
                .cornerRadius(10)
                .shadow(radius: 5)
                .padding(.horizontal)
                .padding(.bottom, 10)  // 画面の下部に余白を追加
            }
            
            VStack {
                Spacer()
                
                // レイヤーボタン、コンパスボタン、現在地ボタンを横に並べる
                HStack {
                    Button(action: {
                        mapViewModel.toggleMapType()
                    }) {
                        Image(systemName: "square.2.layers.3d")
                            .padding()
                            .background(Color.white)
                            .cornerRadius(45)
                            .shadow(radius: 5)
                    }
                    
                    // コンパスボタンの追加
                    CompassView(mapViewModel: mapViewModel)
                        .frame(width: 45, height: 45)
                        .background(Color.white)
                        .cornerRadius(45)
                        .shadow(radius: 5)
                    
                    Button(action: {
                        mapViewModel.goToUserLocation()
                    }) {
                        Image(systemName: "location")
                            .padding()
                            .background(Color.white)
                            .cornerRadius(45)
                            .shadow(radius: 5)
                    }
                }
                .padding(.bottom, 70)  // 検索バーとバランスを取るために余白を追加
            }
        }
        .onAppear {
            mapViewModel.checkLocationAuthorization()
        }
    }
}

// MapViewのSwiftUIラッパー
struct MapView: UIViewRepresentable {
    @ObservedObject var mapViewModel: MapViewModel
    
    func makeUIView(context: Context) -> MKMapView {
        let mapView = mapViewModel.mapView
        mapView.delegate = context.coordinator
        
        // コンパスを表示
        mapView.showsCompass = true
        
        return mapView
    }
    
    func updateUIView(_ uiView: MKMapView, context: Context) {}
    
    // CoordinatorパターンでMKMapViewDelegateを使用
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
    
    class Coordinator: NSObject, MKMapViewDelegate {
        var parent: MapView
        
        init(_ parent: MapView) {
            self.parent = parent
        }
        
        // ユーザーがマップを操作した際に呼ばれる
        func mapView(_ mapView: MKMapView, regionDidChangeAnimated animated: Bool) {
            // ユーザーがマップを動かしたと判断したら、現在地への自動追尾をオフにする
            parent.mapViewModel.isUserInteracting = true
        }
    }
}

// コンパスボタンのラップ
struct CompassView: UIViewRepresentable {
    @ObservedObject var mapViewModel: MapViewModel
    
    func makeUIView(context: Context) -> UIView {
        let compassButton = MKCompassButton(mapView: mapViewModel.mapView)
        compassButton.compassVisibility = .visible
        return compassButton
    }
    
    func updateUIView(_ uiView: UIView, context: Context) {}
}

// MapViewModel
class MapViewModel: NSObject, ObservableObject, CLLocationManagerDelegate {
    @Published var mapView = MKMapView()
    private var locationManager = CLLocationManager()
    private var satelliteViewEnabled = false
    @Published var searchText = ""
    var isUserInteracting = false // ユーザーが手動で操作しているかどうかを判断するフラグ
    
    override init() {
        super.init()
        mapView.showsUserLocation = true
        mapView.userTrackingMode = .follow
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
    }
    
    // 位置情報の権限チェック
    func checkLocationAuthorization() {
        if CLLocationManager.locationServicesEnabled() {
            locationManager.requestWhenInUseAuthorization()
            locationManager.startUpdatingLocation()
        } else {
            print("位置情報サービスが無効です。")
        }
    }
    
    // 位置情報が更新されたときの処理
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        if let userLocation = locations.first {
            // ユーザーが手動でマップを移動させていない場合のみ、現在地に移動
            if !isUserInteracting {
                let region = MKCoordinateRegion(center: userLocation.coordinate, latitudinalMeters: 1000, longitudinalMeters: 1000)
                mapView.setRegion(region, animated: true)
            }
        }
    }
    
    // レイヤー切り替え
    func toggleMapType() {
        satelliteViewEnabled.toggle()
        mapView.mapType = satelliteViewEnabled ? .satellite : .standard
    }
    
    // 現在地に飛ぶ機能
    func goToUserLocation() {
        if let userLocation = locationManager.location {
            let region = MKCoordinateRegion(center: userLocation.coordinate, latitudinalMeters: 1000, longitudinalMeters: 1000)
            mapView.setRegion(region, animated: true)
            isUserInteracting = false // 現在地ボタンを押した場合は再度追尾を有効に
        }
    }
    
    // 場所の検索処理
    func searchLocation() {
        let searchRequest = MKLocalSearch.Request()
        searchRequest.naturalLanguageQuery = searchText
        
        let search = MKLocalSearch(request: searchRequest)
        search.start { response, error in
            guard let response = response else {
                print("エラー: \(String(describing: error?.localizedDescription))")
                return
            }
            
            if let mapItem = response.mapItems.first {
                let coordinate = mapItem.placemark.coordinate
                let region = MKCoordinateRegion(center: coordinate, latitudinalMeters: 1000, longitudinalMeters: 1000)
                self.mapView.setRegion(region, animated: true)
                self.isUserInteracting = true // 検索で移動した場合は追尾を無効に
            }
        }
    }
}
