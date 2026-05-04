# outfit-picker
import SwiftUI

// MARK: - WEATHER DATA MODEL
enum WeatherCondition: String, CaseIterable, Codable {
    case sunny = "Sunny"
    case rainy = "Rainy"
    case cold = "Cold"
    case any = "All Weather"
    
    var icon: String {
        switch self {
        case .sunny: return "sun.max.fill"
        case .rainy: return "cloud.rain.fill"
        case .cold: return "snowflake"
        case .any: return "cloud.sun.fill"
        }
    }
}

// MARK: - THE TRANSLATOR
extension Color {
    init(hex: String) {
        let hex = hex.trimmingCharacters(in: CharacterSet.alphanumerics.inverted)
        var int: UInt64 = 0
        Scanner(string: hex).scanHexInt64(&int)
        let a, r, g, b: UInt64
        switch hex.count {
        case 3: (a, r, g, b) = (255, (int >> 8) * 17, (int >> 4 & 0xF) * 17, (int & 0xF) * 17)
        case 6: (a, r, g, b) = (255, int >> 16, int >> 8 & 0xFF, int & 0xFF)
        case 8: (a, r, g, b) = (int >> 24, int >> 16 & 0xFF, int >> 8 & 0xFF, int & 0xFF)
        default: (a, r, g, b) = (1, 1, 1, 0)
        }
        self.init(.sRGB, red: Double(r) / 255, green: Double(g) / 255, blue: Double(b) / 255, opacity: Double(a) / 255)
    }
    
    func toHex() -> String {
        let components = UIColor(self).cgColor.components
        let r: CGFloat = components?[0] ?? 0.0
        let g: CGFloat = components?[1] ?? 0.0
        let b: CGFloat = components?[2] ?? 0.0
        return String(format: "#%02lX%02lX%02lX", lroundf(Float(r * 255)), lroundf(Float(g * 255)), lroundf(Float(b * 255)))
    }
}

// MARK: - THEME CONFIGURATION
struct AppTheme {
    static let primaryAccent = Color(hex: "#2E0854")
    static let secondaryAccent = Color(hex: "#9FE2BF")
    static let background = Color(hex: "#D1E8FF")
    static let cardBackground = Color(hex: "#CCCCFF")
    static let buttonGradient = LinearGradient(
        gradient: Gradient(colors: [Color(hex: "#9FE2BF"), Color(hex: "#00D2FF"), Color(hex: "#FF0080")]),
        startPoint: .leading, endPoint: .trailing
    )
}

// MARK: - 1. DATA MODEL
enum Category: String, CaseIterable, Codable {
    case jacket = "Jackets"
    case top = "Tops"
    case dress = "Dresses"
    case bottom = "Bottoms"
    case shoes = "Shoes"
    case accessory = "Accessories"
}

struct OutfitItem: Identifiable, Codable {
    var id = UUID()
    var name: String
    var category: Category
    var imageName: String
    var colorHex: String
    var weather: WeatherCondition = .any // NEW: Weather property
}

// MARK: - 2. VIEW MODEL
class WardrobeViewModel: ObservableObject {
    @Published var items: [OutfitItem] = [] { didSet { saveItems() } }
    @Published var selectedWeather: WeatherCondition = .sunny
    
    @Published var selectedTop: OutfitItem?
    @Published var selectedJacket: OutfitItem?
    @Published var selectedDress: OutfitItem?
    @Published var selectedBottom: OutfitItem?
    @Published var selectedShoes: OutfitItem?

    init() {
        loadItems()
        pickRandomOutfit()
    }

    func pickRandomOutfit() {
        // Filter items based on selected weather
        let weatherFilteredItems = items.filter { $0.weather == selectedWeather || $0.weather == .any }
        
        let jackets = weatherFilteredItems.filter { $0.category == .jacket }
        let tops = weatherFilteredItems.filter { $0.category == .top }
        let dresses = weatherFilteredItems.filter { $0.category == .dress }
        let bottoms = weatherFilteredItems.filter { $0.category == .bottom }
        let shoes = weatherFilteredItems.filter { $0.category == .shoes }
        
        selectedJacket = jackets.randomElement()
        
        if !dresses.isEmpty && (tops.isEmpty || Bool.random()) {
            selectedDress = dresses.randomElement()
            selectedTop = nil
            selectedBottom = nil
        } else {
            selectedDress = nil
            selectedTop = tops.randomElement()
            selectedBottom = bottoms.randomElement()
        }
        
        selectedShoes = shoes.randomElement()
    }
    
    func addItem(name: String, category: Category, color: Color, weather: WeatherCondition) {
        let newItem = OutfitItem(name: name, category: category, imageName: "tag.fill", colorHex: color.toHex(), weather: weather)
        items.append(newItem)
    }

    func updateItem(_ updatedItem: OutfitItem) {
        if let index = items.firstIndex(where: { $0.id == updatedItem.id }) {
            items[index] = updatedItem
        }
    }
    
    private func saveItems() {
        if let encoded = try? JSONEncoder().encode(items) {
            UserDefaults.standard.set(encoded, forKey: "SavedWardrobe")
        }
    }

    private func loadItems() {
        if let data = UserDefaults.standard.data(forKey: "SavedWardrobe"),
           let decoded = try? JSONDecoder().decode([OutfitItem].self, from: data) {
            self.items = decoded
        } else {
            self.items = [
                OutfitItem(name: "Leather Jacket", category: .jacket, imageName: "tshirt.fill", colorHex: "#000000", weather: .cold),
                OutfitItem(name: "Blue Jeans", category: .bottom, imageName: "pants.fill", colorHex: "#0000FF", weather: .any),
                OutfitItem(name: "White Sneakers", category: .shoes, imageName: "shoe.fill", colorHex: "#808080", weather: .sunny)
            ]
        }
    }
}

// MARK: - 3. MAIN CONTENT VIEW
struct ContentView: View {
    @StateObject private var viewModel = WardrobeViewModel()
    
    var body: some View {
        TabView {
            GeneratorView(viewModel: viewModel)
                .tabItem { Label("Pick", systemImage: "wand.and.stars") }
            
            WardrobeListView(viewModel: viewModel)
                .tabItem { Label("Wardrobe", systemImage: "closet") }
            
            ShopInspirationView()
                .tabItem { Label("Shop", systemImage: "bag.badge.plus") }
        }
        .accentColor(AppTheme.primaryAccent)
    }
}

// MARK: - 4. GENERATOR VIEW
struct GeneratorView: View {
    @ObservedObject var viewModel: WardrobeViewModel
    
    var body: some View {
        NavigationView {
            ZStack {
                AppTheme.background.ignoresSafeArea()
                
                ScrollView {
                    VStack(spacing: 20) {
                        
                        // NEW: WEATHER PICKER
                        VStack(alignment: .leading) {
                            Text("Current Weather").font(.caption).bold().foregroundColor(AppTheme.primaryAccent).padding(.leading, 5)
                            Picker("Weather", selection: $viewModel.selectedWeather) {
                                ForEach(WeatherCondition.allCases, id: \.self) { condition in
                                    Image(systemName: condition.icon).tag(condition)
                                }
                            }
                            .pickerStyle(SegmentedPickerStyle())
                            .background(Color.white.opacity(0.3))
                            .cornerRadius(8)
                        }
                        .padding(.horizontal)
                        
                        VStack(spacing: 12) {
                            if let jacket = viewModel.selectedJacket {
                                OutfitSlot(label: "Jacket", item: jacket)
                            }
                            if let top = viewModel.selectedTop {
                                OutfitSlot(label: "Top", item: top)
                            }
                            if let dress = viewModel.selectedDress {
                                OutfitSlot(label: "Dress", item: dress)
                            }
                            if let bottom = viewModel.selectedBottom {
                                OutfitSlot(label: "Bottom", item: bottom)
                            }
                            if let shoes = viewModel.selectedShoes {
                                OutfitSlot(label: "Shoes", item: shoes)
                            }
                        }
                        
                        Button(action: {
                            withAnimation(.interpolatingSpring(stiffness: 100, damping: 10)) {
                                viewModel.pickRandomOutfit()
                            }
                        }) {
                            HStack {
                                Image(systemName: "shuffle")
                                Text("Style Me for the \(viewModel.selectedWeather.rawValue)")
                            }
                            .font(.headline).foregroundColor(.white).padding()
                            .frame(maxWidth: .infinity)
                            .background(AppTheme.buttonGradient)
                            .cornerRadius(15)
                        }
                        .padding(.top, 10)
                        
                        Spacer()
                    }
                    .padding()
                }
            }
            .navigationTitle("Daily Pick")
        }
    }
}

// MARK: - 5. WARDROBE LIST VIEW
struct WardrobeListView: View {
    @ObservedObject var viewModel: WardrobeViewModel
    @State private var showingAddSheet = false
    @State private var itemToEdit: OutfitItem?

    var body: some View {
        NavigationView {
            List {
                ForEach(Category.allCases.filter { cat in viewModel.items.contains(where: { $0.category == cat }) }, id: \.self) { category in
                    Section(header: Text(category.rawValue).font(.subheadline).bold()) {
                        ForEach(viewModel.items.filter { $0.category == category }) { item in
                            Button(action: { itemToEdit = item }) {
                                HStack {
                                    Circle().fill(Color(hex: item.colorHex)).frame(width: 12, height: 12)
                                    Image(systemName: item.weather.icon).foregroundColor(.gray).font(.caption) // Weather Icon
                                    Text(item.name).font(.body).foregroundColor(.primary)
                                    Spacer()
                                    Image(systemName: "pencil").font(.caption).foregroundColor(.gray)
                                }
                            }
                        }
                        .onDelete { indexSet in
                            let itemsInCategory = viewModel.items.filter { $0.category == category }
                            let itemsToDelete = indexSet.map { itemsInCategory[$0] }
                            viewModel.items.removeAll { item in itemsToDelete.contains { $0.id == item.id } }
                        }
                    }
                }
            }
            .navigationTitle("My Wardrobe")
            .toolbar {
                Button(action: { showingAddSheet = true }) {
                    Image(systemName: "plus.circle.fill").font(.title3)
                }
            }
            .sheet(isPresented: $showingAddSheet) { AddItemView(viewModel: viewModel) }
            .sheet(item: $itemToEdit) { item in
                EditItemView(viewModel: viewModel, item: item)
            }
        }
    }
}

// MARK: - 6. ADD ITEM VIEW
struct AddItemView: View {
    @Environment(\.dismiss) var dismiss
    @ObservedObject var viewModel: WardrobeViewModel
    @State private var name = ""
    @State private var category = Category.top
    @State private var selectedColor = Color.blue
    @State private var weather = WeatherCondition.any
    
    var body: some View {
        NavigationView {
            Form {
                Section(header: Text("Item Details")) {
                    TextField("Name (e.g. Red Hoodie)", text: $name)
                    Picker("Category", selection: $category) {
                        ForEach(Category.allCases, id: \.self) { Text($0.rawValue).tag($0) }
                    }
                }
                Section(header: Text("Best for Weather")) {
                    Picker("Weather Type", selection: $weather) {
                        ForEach(WeatherCondition.allCases, id: \.self) { condition in
                            Label(condition.rawValue, systemImage: condition.icon).tag(condition)
                        }
                    }
                }
                Section(header: Text("Style")) { ColorPicker("Select Color", selection: $selectedColor) }
                
                Button("Add to Closet") {
                    viewModel.addItem(name: name, category: category, color: selectedColor, weather: weather)
                    dismiss()
                }
                .disabled(name.isEmpty)
                .frame(maxWidth: .infinity)
                .font(.headline)
            }
            .navigationTitle("New Clothing")
            .toolbar { Button("Cancel") { dismiss() } }
        }
    }
}

// MARK: - 6b. EDIT ITEM VIEW
struct EditItemView: View {
    @Environment(\.dismiss) var dismiss
    @ObservedObject var viewModel: WardrobeViewModel
    let item: OutfitItem
    
    @State private var name: String
    @State private var category: Category
    @State private var selectedColor: Color
    @State private var weather: WeatherCondition

    init(viewModel: WardrobeViewModel, item: OutfitItem) {
        self.viewModel = viewModel
        self.item = item
        _name = State(initialValue: item.name)
        _category = State(initialValue: item.category)
        _selectedColor = State(initialValue: Color(hex: item.colorHex))
        _weather = State(initialValue: item.weather)
    }


    In this app you can get outfit help for any weather you choose a weather type,press styleme and you get your outfit.You also add new items with the category,weather type and color.You can also buy new things to add to your closet it add to a pervious outfit.The only issue i had was adding new code to my already existing code i would add it in and would get a bunch of errors so i would have to go bac and have to AI add in the code for me.

    var body: some View {
        NavigationView {
            Form {
                Section(header: Text("Item Details")) {
                    TextField("Name", text: $name)
                    Picker("Category", selection: $category) {
                        ForEach(Category.allCases, id: \.self) { Text($0.rawValue).tag($0) }
                    }
                }
                Section(header: Text("Weather")) {
                    Picker("Weather Type", selection: $weather) {
                        ForEach(WeatherCondition.allCases, id: \.self) { condition in
                            Label(condition.rawValue, systemImage: condition.icon).tag(condition)
                        }
                    }
                }
                Section(header: Text("Style")) { ColorPicker("Change Color", selection: $selectedColor) }
                
                Button("Save Changes") {
                    let updated = OutfitItem(id: item.id, name: name, category: category, imageName: item.imageName, colorHex: selectedColor.toHex(), weather: weather)
                    viewModel.updateItem(updated)
                    dismiss()
                }
                .frame(maxWidth: .infinity)
                .font(.headline)
            }
            .navigationTitle("Edit Item")
            .toolbar { Button("Cancel") { dismiss() } }
        }
    }
}

// MARK: - 7. SHOP INSPIRATION VIEW
struct ShopInspirationView: View {
    let trendingItems = ["White Linen Shirt", "Straight Leg Jeans", "Chelsea Boots", "Gold Hoop Earrings", "Beige Trench Coat"]
    var body: some View {
        NavigationView {
            ZStack {
                AppTheme.background.ignoresSafeArea()
                List {
                    Section(header: Text("Trending Pieces")) {
                        ForEach(trendingItems, id: \.self) { trend in
                            Link(destination: URL(string: "https://www.google.com/search?q=buy+\(trend.replacingOccurrences(of: " ", with: "+"))")!) {
                                HStack {
                                    VStack(alignment: .leading) {
                                        Text(trend).font(.headline).foregroundColor(.primary)
                                        Text("Find a version of this").font(.caption).foregroundColor(.secondary)
                                    }
                                    Spacer()
                                    Image(systemName: "arrow.up.right.square.fill").foregroundColor(AppTheme.secondaryAccent)
                                }
                            }
                        }
                    }
                }
                .listStyle(InsetGroupedListStyle())
            }
            .navigationTitle("Inspiration")
        }
    }
}

// MARK: - 8. SUPPORTING UI COMPONENTS
struct OutfitSlot: View {
    let label: String
    let item: OutfitItem?
    
    var body: some View {
        HStack {
            VStack(alignment: .leading) {
                Text(label.uppercased()).font(.caption2).bold().foregroundColor(AppTheme.primaryAccent)
                Text(item?.name ?? "Tap shuffle...").font(.body).fontWeight(.medium)
            }
            Spacer()
            if let item = item {
                ZStack {
                    Circle().fill(Color(hex: item.colorHex).opacity(0.15)).frame(width: 44, height: 44)
                    Image(systemName: item.imageName).font(.system(size: 24)).foregroundColor(Color(hex: item.colorHex))
                }
            }
        }
        .padding().frame(height: 80).background(AppTheme.cardBackground).cornerRadius(18)
    }
}

// MARK: - PREVIEW
struct ContentView_Previews: PreviewProvider {
    static var previews: some View { ContentView() }


    For this app is to help pick an outfit for weather pick weather type click style me and you got your outift.You can also add now clothes pick a category,weather and color and add it to your closet.You can also buy new things to add to your wardrobe and buy them for later .The ony issue was adding new code and putting it in to my already existing code and i would get error so i had AI redo it for me again.
}
