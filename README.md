# Handschrift.hmtlimport SwiftUI
import PencilKit

// MARK: - Haupt-App
@main
struct HandwritingFontApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

// MARK: - Datenmodell
class LetterStore: ObservableObject {
    @Published var letters: [Character: UIImage] = [:]
    
    func save(letter: Character, drawing: PKDrawing, size: CGSize) {
        // Zeichnung in Bild umwandeln (mit etwas Padding)
        let bounds = drawing.bounds.insetBy(dx: -20, dy: -20)
        let image = drawing.image(from: bounds, scale: UIScreen.main.scale)
        letters[letter] = image
    }
    
    func image(for char: Character) -> UIImage? {
        letters[char]
    }
}

// MARK: - Canvas Wrapper
struct PencilCanvas: UIViewRepresentable {
    @Binding var canvasView: PKCanvasView
    var onDrawingChanged: (() -> Void)? = nil
    
    func makeUIView(context: Context) -> PKCanvasView {
        canvasView.tool = PKInkingTool(.pen, color: .black, width: 4)
        canvasView.drawingPolicy = .pencilOnly          // nur Apple Pencil
        canvasView.backgroundColor = .systemGray6
        canvasView.isOpaque = false
        return canvasView
    }
    
    func updateUIView(_ uiView: PKCanvasView, context: Context) {}
}

// MARK: - Haupt-View
struct ContentView: View {
    @StateObject private var store = LetterStore()
    @State private var canvasView = PKCanvasView()
    @State private var selectedChar: Character = "A"
    @State private var inputText = "Hallo Welt"
    @State private var showingPreview = false
    
    // Alle Zeichen, die man zeichnen kann
    let characters: [Character] = Array("ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789 .,!?")
    
    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                
                // 1. Buchstaben-Auswahl
                ScrollView(.horizontal, showsIndicators: false) {
                    HStack {
                        ForEach(characters, id: \.self) { char in
                            Button {
                                selectedChar = char
                                canvasView.drawing = PKDrawing() // Canvas leeren
                            } label: {
                                Text(String(char))
                                    .font(.title2.bold())
                                    .frame(width: 44, height: 44)
                                    .background(selectedChar == char ? Color.blue : Color.gray.opacity(0.2))
                                    .foregroundStyle(selectedChar == char ? .white : .primary)
                                    .clipShape(RoundedRectangle(cornerRadius: 8))
                                    .overlay {
                                        if store.letters[char] != nil {
                                            Circle()
                                                .fill(.green)
                                                .frame(width: 10, height: 10)
                                                .offset(x: 16, y: -16)
                                        }
                                    }
                            }
                        }
                    }
                    .padding(.horizontal)
                }
                
                Text("Zeichne den Buchstaben: \(selectedChar)")
                    .font(.headline)
                
                // 2. Zeichenfläche
                PencilCanvas(canvasView: $canvasView)
                    .frame(height: 220)
                    .clipShape(RoundedRectangle(cornerRadius: 12))
                    .overlay(
                        RoundedRectangle(cornerRadius: 12)
                            .stroke(Color.gray.opacity(0.4), lineWidth: 1)
                    )
                    .padding(.horizontal)
                
                // 3. Speichern / Löschen
                HStack {
                    Button("Löschen") {
                        canvasView.drawing = PKDrawing()
                    }
                    .buttonStyle(.bordered)
                    
                    Spacer()
                    
                    Button("Buchstaben speichern") {
                        store.save(letter: selectedChar,
                                   drawing: canvasView.drawing,
                                   size: canvasView.bounds.size)
                        // Optional: nächstes Zeichen auswählen
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding(.horizontal)
                
                Divider()
                
                // 4. Text eingeben und umwandeln
                TextField("Text eingeben...", text: $inputText)
                    .textFieldStyle(.roundedBorder)
                    .padding(.horizontal)
                
                Button("In Handschrift umwandeln") {
                    showingPreview = true
                }
                .buttonStyle(.borderedProminent)
                .disabled(inputText.isEmpty)
                
                Spacer()
            }
            .navigationTitle("Meine Handschrift")
            .sheet(isPresented: $showingPreview) {
                HandwritingPreview(text: inputText, store: store)
            }
        }
    }
}

// MARK: - Vorschau der zusammengesetzten Handschrift
struct HandwritingPreview: View {
    let text: String
    @ObservedObject var store: LetterStore
    @Environment(\.dismiss) private var dismiss
    
    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 12) {
                    // Die zusammengesetzte Handschrift
                    HandwrittenTextView(text: text, store: store)
                        .padding()
                    
                    // Zum Vergleich normaler Text
                    Text("Original: \(text)")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                        .padding(.horizontal)
                }
            }
            .navigationTitle("Ergebnis")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .topBarTrailing) {
                    Button("Fertig") { dismiss() }
                }
            }
        }
    }
}

// MARK: - View, die den Text aus den Buchstaben-Bildern zusammensetzt
struct HandwrittenTextView: View {
    let text: String
    @ObservedObject var store: LetterStore
    
    var body: some View {
        // Wir bauen eine Zeile aus den Bildern
        // (einfache Version – für mehrzeilig müsste man noch umbrechen)
        HStack(alignment: .bottom, spacing: 2) {
            ForEach(Array(text), id: \.self) { char in
                if let image = store.image(for: char) {
                    Image(uiImage: image)
                        .resizable()
                        .scaledToFit()
                        .frame(height: 60) // feste Höhe, Breite proportional
                } else if char == " " {
                    // Leerzeichen
                    Color.clear
                        .frame(width: 20, height: 60)
                } else {
                    // Fallback, falls Buchstabe noch nicht gezeichnet
                    Text(String(char))
                        .font(.system(size: 40, design: .serif))
                        .foregroundStyle(.red.opacity(0.6))
                        .frame(height: 60)
                }
            }
        }
        .padding()
        .background(Color.white)
        .clipShape(RoundedRectangle(cornerRadius: 8))
        .shadow(radius: 2)
    }
}
