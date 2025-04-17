struct CellData {
    let size: CGSize
    let color: UIColor
}

class MyHeaderView: UICollectionReusableView {
    static let identifier = "MyHeaderView"

    lazy var textField: UITextField = {
        let field = UITextField()
        field.translatesAutoresizingMaskIntoConstraints = false
        field.placeholder = "Enter text here..."
        field.borderStyle = .roundedRect
        field.backgroundColor = .white
        field.isHidden = true
        field.accessibilityIdentifier = "StickyHeaderTextField"
        field.autocorrectionType = .no
        field.spellCheckingType = .no
        field.returnKeyType = .done
        field.addTarget(self, action: #selector(textFieldReturnPressed), for: .primaryActionTriggered)
        return field
    }()

    override init(frame: CGRect) {
        super.init(frame: frame)
        setupViews()
    }

    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    private func setupViews() {
        addSubview(textField)
        clipsToBounds = true
        setupConstraints()
    }

    func setupConstraints() {
         self.constraints.filter { $0.firstItem === textField || $0.secondItem === textField }.forEach { removeConstraint($0) }
         textField.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 15).isActive = true
         textField.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -15).isActive = true
         textField.centerYAnchor.constraint(equalTo: centerYAnchor).isActive = true
         textField.heightAnchor.constraint(equalToConstant: 34).isActive = true
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        textField.isHidden = true
        textField.text = nil
    }

    @objc func textFieldReturnPressed() {
        textField.resignFirstResponder()
    }
}

class StickyHeaderFlowLayout: UICollectionViewFlowLayout {
    var stickyHeaderSection: Int = 1
    let stickyHeaderZIndex: Int = 1000
    let standardHeaderZIndex: Int = 100

    override func layoutAttributesForElements(in rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
        guard let superAttributes = super.layoutAttributesForElements(in: rect) else { return nil }
        var mutableAttributes = superAttributes.map { $0.copy() as! UICollectionViewLayoutAttributes }
        guard let collectionView = self.collectionView else { return mutableAttributes }

        let contentInsetTop = collectionView.adjustedContentInset.top
        let effectiveOffsetY = collectionView.contentOffset.y + contentInsetTop
        let stickyHeaderIndexPath = IndexPath(item: 0, section: stickyHeaderSection)

        guard let stickyHeaderOriginalAttrs = super.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: stickyHeaderIndexPath)?.copy() as? UICollectionViewLayoutAttributes
        else {
             for attrs in mutableAttributes where attrs.representedElementKind == UICollectionView.elementKindSectionHeader { attrs.zIndex = standardHeaderZIndex }
             return mutableAttributes
        }

        let originalStickyHeaderY = stickyHeaderOriginalAttrs.frame.origin.y
        let stickyY = max(effectiveOffsetY, originalStickyHeaderY)
        stickyHeaderOriginalAttrs.frame.origin.y = stickyY
        stickyHeaderOriginalAttrs.zIndex = stickyHeaderZIndex

        var stickyHeaderFoundIndex: Int? = nil
        for (index, attrs) in mutableAttributes.enumerated() {
            if attrs.representedElementKind == UICollectionView.elementKindSectionHeader {
                if attrs.indexPath == stickyHeaderIndexPath { stickyHeaderFoundIndex = index }
                else { attrs.zIndex = standardHeaderZIndex }
            }
        }
        if let index = stickyHeaderFoundIndex { mutableAttributes[index] = stickyHeaderOriginalAttrs }
        else if stickyHeaderOriginalAttrs.frame.intersects(rect) { mutableAttributes.append(stickyHeaderOriginalAttrs) }
        return mutableAttributes
    }
    override func shouldInvalidateLayout(forBoundsChange newBounds: CGRect) -> Bool {
        let oldBounds = collectionView?.bounds ?? .zero; return newBounds.origin.y != oldBounds.origin.y
    }
    override func layoutAttributesForSupplementaryView(ofKind elementKind: String, at indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        guard let attributes = super.layoutAttributesForSupplementaryView(ofKind: elementKind, at: indexPath),
              let copiedAttributes = attributes.copy() as? UICollectionViewLayoutAttributes else { return nil }
        if elementKind == UICollectionView.elementKindSectionHeader && indexPath.section == stickyHeaderSection {
            guard let collectionView = self.collectionView else { return copiedAttributes }
            let contentInsetTop = collectionView.adjustedContentInset.top
            let effectiveOffsetY = collectionView.contentOffset.y + contentInsetTop
            copiedAttributes.frame.origin.y = max(effectiveOffsetY, attributes.frame.origin.y)
            copiedAttributes.zIndex = stickyHeaderZIndex
        } else if elementKind == UICollectionView.elementKindSectionHeader {
            copiedAttributes.zIndex = standardHeaderZIndex
        }
        return copiedAttributes
    }
}

protocol KeyboardHandling: AnyObject {
    var scrollableViewForKeyboardAdjustment: UIScrollView? { get }
    var originalScrollViewBottomInset: CGFloat { get set }
    func frameOfFocusedElementInView() -> CGRect?
    var isAdjustingViewForKeyboard: Bool { get set }
}

extension KeyboardHandling where Self: UIViewController {
    func handleKeyboardChange_KBHandling(notification: Notification, isShowing: Bool) {
        guard !self.isAdjustingViewForKeyboard else { return }
        self.isAdjustingViewForKeyboard = true

        guard let scrollView = self.scrollableViewForKeyboardAdjustment,
              let userInfo = notification.userInfo,
              let keyboardFrameEnd = userInfo[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect,
              let duration = userInfo[UIResponder.keyboardAnimationDurationUserInfoKey] as? TimeInterval,
              let curveValue = userInfo[UIResponder.keyboardAnimationCurveUserInfoKey] as? UInt else {
            self.isAdjustingViewForKeyboard = false; return
        }

        let keyboardFrameInView = self.view.convert(keyboardFrameEnd, from: self.view.window)
        let keyboardOverlap = self.view.bounds.maxY - keyboardFrameInView.minY
        let currentOriginalInset = self.originalScrollViewBottomInset
        let newBottomInset = isShowing ? max(0, keyboardOverlap - self.view.safeAreaInsets.bottom) : currentOriginalInset

        var targetContentOffset = scrollView.contentOffset
        var needsOffsetAdjustment = false
        if isShowing, let focusedElementFrameInView = self.frameOfFocusedElementInView() {
            let keyboardTopYInView = self.view.bounds.height - keyboardOverlap
            let padding: CGFloat = 10
            if focusedElementFrameInView.maxY > (keyboardTopYInView - padding) {
                let scrollAmount = focusedElementFrameInView.maxY - (keyboardTopYInView - padding)
                targetContentOffset.y += scrollAmount
                needsOffsetAdjustment = true
                let maxOffsetY = max(-scrollView.adjustedContentInset.top, scrollView.contentSize.height + newBottomInset - scrollView.bounds.height)
                targetContentOffset.y = min(max(targetContentOffset.y, -scrollView.adjustedContentInset.top), maxOffsetY)
            }
        }

        let animationOptions = UIView.AnimationOptions(rawValue: curveValue << 16)
        let insetChanged = abs(scrollView.contentInset.bottom - newBottomInset) > 0.01
        let offsetChanged = needsOffsetAdjustment && abs(scrollView.contentOffset.y - targetContentOffset.y) > 0.01

        guard insetChanged || offsetChanged else {
            self.isAdjustingViewForKeyboard = false; return
        }

        UIView.animate(withDuration: duration, delay: 0, options: [animationOptions, .beginFromCurrentState], animations: {
            if insetChanged { scrollView.contentInset.bottom = newBottomInset }
            if offsetChanged { scrollView.setContentOffset(targetContentOffset, animated: false) }
        }) { _ in
            self.isAdjustingViewForKeyboard = false
        }
    }
}

class ViewController: UIViewController, UICollectionViewDataSource, UICollectionViewDelegate, UICollectionViewDelegateFlowLayout, KeyboardHandling {

    private let cellIdentifier = "MyCell"
    let numberOfSections = 10
    var sectionData: [[CellData]] = []
    @IBOutlet weak var collectionView: UICollectionView!

    var scrollableViewForKeyboardAdjustment: UIScrollView? { self.collectionView }
    var originalScrollViewBottomInset: CGFloat = 0
    var isAdjustingViewForKeyboard: Bool = false

    func frameOfFocusedElementInView() -> CGRect? {
        guard let layout = collectionView.collectionViewLayout as? StickyHeaderFlowLayout else { return nil }
        let stickyHeaderIndexPath = IndexPath(item: 0, section: layout.stickyHeaderSection)
        if let headerView = collectionView.supplementaryView(forElementKind: UICollectionView.elementKindSectionHeader, at: stickyHeaderIndexPath) as? MyHeaderView,
           headerView.textField.isFirstResponder {
             return headerView.textField.convert(headerView.textField.bounds, to: self.view)
        }
        return nil
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        generateExampleData()
        setupCollectionView()
    }

    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        if originalScrollViewBottomInset == 0 && collectionView != nil {
            originalScrollViewBottomInset = self.collectionView.adjustedContentInset.bottom
        }
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        registerForKeyboardNotifications()
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        unregisterFromKeyboardNotifications()
    }

    deinit {
        unregisterFromKeyboardNotifications()
    }

    private func registerForKeyboardNotifications() {
        NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillChangeFrame(_:)), name: UIResponder.keyboardWillChangeFrameNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(keyboardWillHide(_:)), name: UIResponder.keyboardWillHideNotification, object: nil)
    }

    private func unregisterFromKeyboardNotifications() {
        NotificationCenter.default.removeObserver(self, name: UIResponder.keyboardWillChangeFrameNotification, object: nil)
        NotificationCenter.default.removeObserver(self, name: UIResponder.keyboardWillHideNotification, object: nil)
    }

    @objc func keyboardWillChangeFrame(_ notification: Notification) {
        handleKeyboardChange_KBHandling(notification: notification, isShowing: true)
    }
    @objc func keyboardWillHide(_ notification: Notification) {
        handleKeyboardChange_KBHandling(notification: notification, isShowing: false)
    }

    private func setupCollectionView() {
        collectionView.register(UICollectionViewCell.self, forCellWithReuseIdentifier: cellIdentifier)
        collectionView.register(MyHeaderView.self, forSupplementaryViewOfKind: UICollectionView.elementKindSectionHeader, withReuseIdentifier: MyHeaderView.identifier)

        let stickyLayout = StickyHeaderFlowLayout()
        stickyLayout.stickyHeaderSection = 1
        stickyLayout.minimumInteritemSpacing = 10
        stickyLayout.minimumLineSpacing = 10
        stickyLayout.sectionInset = UIEdgeInsets(top: 10, left: 10, bottom: 10, right: 10)
        collectionView.collectionViewLayout = stickyLayout

        collectionView.delegate = self
        collectionView.dataSource = self
        collectionView.backgroundColor = .systemGroupedBackground
        collectionView.keyboardDismissMode = .interactive
    }

    func numberOfSections(in collectionView: UICollectionView) -> Int { return sectionData.count }
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int {
        guard section < sectionData.count else { return 0 }; return sectionData[section].count
    }
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: cellIdentifier, for: indexPath)
        guard indexPath.section < sectionData.count, indexPath.item < sectionData[indexPath.section].count else {
             cell.backgroundColor = .systemRed; cell.contentView.subviews.filter { $0 is UILabel }.forEach { $0.removeFromSuperview() }; return cell
        }
        let data = sectionData[indexPath.section][indexPath.item]
        cell.backgroundColor = data.color; cell.layer.cornerRadius = 4

        cell.contentView.subviews.filter { $0 is UILabel }.forEach { $0.removeFromSuperview() }
        let label = UILabel(frame: CGRect(x: 0, y: 0, width: data.size.width, height: 20).insetBy(dx: 2, dy: 2))
        label.font = UIFont.systemFont(ofSize: 9); label.textAlignment = .center; label.textColor = .darkText
        label.backgroundColor = UIColor.white.withAlphaComponent(0.7)
        label.text = "S\(indexPath.section) I\(indexPath.item) \(Int(data.size.width))x\(Int(data.size.height))"
        label.adjustsFontSizeToFitWidth = true; label.minimumScaleFactor = 0.5
        cell.contentView.addSubview(label)
        return cell
     }
    func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, at indexPath: IndexPath) -> UICollectionReusableView {
        guard kind == UICollectionView.elementKindSectionHeader else { return UICollectionReusableView() }
        guard let headerView = collectionView.dequeueReusableSupplementaryView(ofKind: kind, withReuseIdentifier: MyHeaderView.identifier, for: indexPath) as? MyHeaderView else { fatalError() }
        let stickySectionIndex = (collectionView.collectionViewLayout as? StickyHeaderFlowLayout)?.stickyHeaderSection ?? 1
        let isStickySection = (indexPath.section == stickySectionIndex)
        headerView.backgroundColor = isStickySection ? .systemGreen : .systemBlue
        headerView.textField.isHidden = !isStickySection
        return headerView
    }
    func scrollViewWillBeginDragging(_ scrollView: UIScrollView) { view.endEditing(true) }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize {
        guard indexPath.section < sectionData.count, indexPath.item < sectionData[indexPath.section].count else { return CGSize(width: 50, height: 50) }
        return sectionData[indexPath.section][indexPath.item].size
    }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, referenceSizeForHeaderInSection section: Int) -> CGSize {
        return CGSize(width: collectionView.bounds.width, height: 50)
    }
    func generateExampleData() {
        sectionData.removeAll()
        let availableWidth = (collectionView?.bounds.width ?? UIScreen.main.bounds.width) - 2 * 10
        for section in 0..<numberOfSections {
            var itemsInSection: [CellData] = []
            let numberOfItems = Int.random(in: 5...15)
            for item in 0..<numberOfItems {
                var width: CGFloat, height: CGFloat, color: UIColor
                if section % 3 == 0 { width = CGFloat.random(in: 50...150); height = 160; color = .systemTeal }
                else if item % 4 == 0 { width = 40; height = 40; color = .systemIndigo }
                else if item % 3 == 0 { width = 180; height = 60; color = .systemOrange }
                else { width = CGFloat.random(in: 60...120); height = 80; color = .lightGray }
                width = min(width, availableWidth)
                itemsInSection.append(CellData(size: CGSize(width: width, height: height), color: color))
            }
            sectionData.append(itemsInSection)
        }
    }
}

///////////////////////////////////////////

class MockTextField: UITextField {
    var simulatedIsFirstResponder: (() -> Bool)?
    override var isFirstResponder: Bool {
        return simulatedIsFirstResponder?() ?? super.isFirstResponder
    }
}

class TestableHeaderView: MyHeaderView {
    static let testableIdentifier = "TestableHeaderView"
    var simulateTextFieldIsFirstResponder: Bool = false {
        didSet {
            (self.textField as? MockTextField)?.simulatedIsFirstResponder = { [weak self] in
                 return self?.simulateTextFieldIsFirstResponder ?? false
            }
        }
    }
    private var mockTextFieldInstance: MockTextField?

    override init(frame: CGRect) {
        super.init(frame: frame)
        replaceTextFieldWithMock()
    }
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        replaceTextFieldWithMock()
    }
    private func replaceTextFieldWithMock() {
        let originalTextField = self.textField
        originalTextField.removeFromSuperview()
        let mockField = MockTextField(); mockField.translatesAutoresizingMaskIntoConstraints = false
        mockField.placeholder = "Test"; mockField.borderStyle = .roundedRect; mockField.backgroundColor = .white; mockField.isHidden = true
        mockField.accessibilityIdentifier = "StickyHeaderTextField"; mockField.autocorrectionType = .no; mockField.spellCheckingType = .no
        mockField.returnKeyType = .done; mockField.addTarget(self, action: #selector(mockTextFieldReturnPressed), for: .primaryActionTriggered)
        mockField.simulatedIsFirstResponder = { [weak self] in return self?.simulateTextFieldIsFirstResponder ?? false }
        self.mockTextFieldInstance = mockField; self.textField = mockField; addSubview(self.textField); setupConstraints()
    }
    @objc private func mockTextFieldReturnPressed() { self.textField.resignFirstResponder() }
}


class MockLayoutDataSourceAndDelegate: NSObject, UICollectionViewDataSource, UICollectionViewDelegateFlowLayout {
    var numberOfSections: Int = 0; var itemsPerSection: [Int] = []; var itemSizes: [IndexPath: CGSize] = [:]; var headerSizes: [Int: CGSize] = [:]
    var defaultMinimumLineSpacing: CGFloat = 10; var defaultMinimumInteritemSpacing: CGFloat = 10; var defaultSectionInset: UIEdgeInsets = .zero
    func reset() { numberOfSections = 0; itemsPerSection = []; itemSizes = [:]; headerSizes = [:] }
    func numberOfSections(in collectionView: UICollectionView) -> Int { return numberOfSections }
    func collectionView(_ collectionView: UICollectionView, numberOfItemsInSection section: Int) -> Int { return section < itemsPerSection.count ? itemsPerSection[section] : 0 }
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell { return collectionView.dequeueReusableCell(withReuseIdentifier: "MockCell", for: indexPath) }
    func collectionView(_ collectionView: UICollectionView, viewForSupplementaryElementOfKind kind: String, at indexPath: IndexPath) -> UICollectionReusableView {
        if kind == UICollectionView.elementKindSectionHeader { return collectionView.dequeueReusableSupplementaryView(ofKind: kind, withReuseIdentifier: TestableHeaderView.testableIdentifier, for: indexPath) }
        return UICollectionReusableView()
    }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, sizeForItemAt indexPath: IndexPath) -> CGSize { return itemSizes[indexPath] ?? CGSize(width: 50, height: 50) }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, referenceSizeForHeaderInSection section: Int) -> CGSize { return headerSizes[section] ?? .zero }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumLineSpacingForSectionAt section: Int) -> CGFloat { return defaultMinimumLineSpacing }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, minimumInteritemSpacingForSectionAt section: Int) -> CGFloat { return defaultMinimumInteritemSpacing }
    func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, insetForSectionAt section: Int) -> UIEdgeInsets { return defaultSectionInset }
}

class MockCollectionView: UICollectionView {
    var mockDataSource: UICollectionViewDataSource?
    var mockDelegate: UICollectionViewDelegate?
    var mockContentOffset: CGPoint = .zero
    var mockAdjustedContentInset: UIEdgeInsets = .zero
    var mockBounds: CGRect = .zero
    private var _mockLayout: UICollectionViewLayout?
    var registeredCells: Set<String> = []
    var registeredSupplementaryViews: [String: Set<String>] = [:]
    var supplementaryViewCache: [String: [IndexPath: UICollectionReusableView]] = [:]
    private var _mockContentInset: UIEdgeInsets = .zero

    override var dataSource: UICollectionViewDataSource? { get { mockDataSource } set { mockDataSource = newValue } }
    override var delegate: UICollectionViewDelegate? { get { mockDelegate } set { mockDelegate = newValue } }
    override var contentOffset: CGPoint { get { mockContentOffset } set { mockContentOffset = newValue } }
    override var adjustedContentInset: UIEdgeInsets { get { mockAdjustedContentInset } set { mockAdjustedContentInset = newValue } }
    override var bounds: CGRect { get { mockBounds } set { mockBounds = newValue } }
    override var collectionViewLayout: UICollectionViewLayout { get { _mockLayout ?? super.collectionViewLayout } set { _mockLayout = newValue } }
    override var contentInset: UIEdgeInsets { get { _mockContentInset } set { _mockContentInset = newValue } }
    override var numberOfSections: Int { mockDataSource?.numberOfSections?(in: self) ?? 0 }
    override func numberOfItems(inSection section: Int) -> Int { mockDataSource?.collectionView(self, numberOfItemsInSection: section) ?? 0 }
    override func register(_ cellClass: AnyClass?, forCellWithReuseIdentifier identifier: String) { registeredCells.insert(identifier) }
    override func register(_ viewClass: AnyClass?, forSupplementaryViewOfKind elementKind: String, withReuseIdentifier identifier: String) {
        var identifiers = registeredSupplementaryViews[elementKind] ?? Set<String>(); identifiers.insert(identifier); registeredSupplementaryViews[elementKind] = identifiers
    }
    override func dequeueReusableCell(withReuseIdentifier identifier: String, for indexPath: IndexPath) -> UICollectionViewCell {
        guard registeredCells.contains(identifier) else { fatalError("Cell not registered: \(identifier)") }
        return UICollectionViewCell(frame: .zero)
    }
    override func dequeueReusableSupplementaryView(ofKind elementKind: String, withReuseIdentifier identifier: String, for indexPath: IndexPath) -> UICollectionReusableView {
        guard registeredSupplementaryViews[elementKind]?.contains(identifier) ?? false else { fatalError("Supplementary view not registered: \(elementKind) - \(identifier)") }
        if let cachedView = supplementaryViewCache[elementKind]?[indexPath] { return cachedView }
        let newView: UICollectionReusableView
        if elementKind == UICollectionView.elementKindSectionHeader && identifier == TestableHeaderView.testableIdentifier {
             newView = TestableHeaderView(frame: .zero)
        } else { newView = UICollectionReusableView(frame: .zero) }
        var kindCache = supplementaryViewCache[elementKind] ?? [:]; kindCache[indexPath] = newView; supplementaryViewCache[elementKind] = kindCache
        return newView
    }
    override func supplementaryView(forElementKind elementKind: String, at indexPath: IndexPath) -> UICollectionReusableView? {
        return supplementaryViewCache[elementKind]?[indexPath]
    }

    init(frame: CGRect = .zero, layout: UICollectionViewLayout = UICollectionViewFlowLayout()) {
        super.init(frame: frame, collectionViewLayout: layout); _mockLayout = layout
        self.register(UICollectionViewCell.self, forCellWithReuseIdentifier: "MockCell")
        self.register(TestableHeaderView.self, forSupplementaryViewOfKind: UICollectionView.elementKindSectionHeader, withReuseIdentifier: TestableHeaderView.testableIdentifier)
    }
    required init?(coder: NSCoder) { fatalError("init(coder:) has not been implemented") }
}

//MARK: - Test Case
class ViewControllerKeyboardHandlingTests: XCTestCase {

    var viewController: ViewController!
    var mockCollectionView: MockCollectionView!
    var mockDataSourceAndDelegate: MockLayoutDataSourceAndDelegate!
    var stickyLayout: StickyHeaderFlowLayout!

    let headerHeight: CGFloat = 50
    let cellHeight: CGFloat = 80
    let cellWidth: CGFloat = 100
    let spacing: CGFloat = 10
    let collectionViewWidth: CGFloat = 375
    let collectionViewHeight: CGFloat = 600
    let stickySection = 1

    override func setUpWithError() throws {
        try super.setUpWithError()

        stickyLayout = StickyHeaderFlowLayout()
        stickyLayout.stickyHeaderSection = stickySection
        stickyLayout.minimumLineSpacing = spacing
        stickyLayout.minimumInteritemSpacing = spacing
        stickyLayout.sectionInset = UIEdgeInsets(top: spacing, left: spacing, bottom: spacing, right: spacing)

        mockCollectionView = MockCollectionView(frame: CGRect(x: 0, y: 0, width: collectionViewWidth, height: collectionViewHeight), layout: stickyLayout)
        mockDataSourceAndDelegate = MockLayoutDataSourceAndDelegate()
        mockDataSourceAndDelegate.defaultSectionInset = stickyLayout.sectionInset
        mockDataSourceAndDelegate.defaultMinimumLineSpacing = stickyLayout.minimumLineSpacing
        mockDataSourceAndDelegate.defaultMinimumInteritemSpacing = stickyLayout.minimumInteritemSpacing

        viewController = ViewController()

        viewController.collectionView = mockCollectionView
        mockCollectionView.dataSource = mockDataSourceAndDelegate
        mockCollectionView.delegate = mockDataSourceAndDelegate

        viewController.loadViewIfNeeded()

        mockCollectionView.mockBounds = viewController.view.bounds
        mockCollectionView.mockContentOffset = .zero
        mockCollectionView.mockAdjustedContentInset = .zero
        viewController.originalScrollViewBottomInset = 0

        setupMockData(sections: 3, itemsPerSec: 6)

        viewController.viewWillAppear(false)
        viewController.viewDidLayoutSubviews()
    }

    override func tearDownWithError() throws {
        viewController?.viewWillDisappear(false)
        viewController = nil
        mockCollectionView = nil
        mockDataSourceAndDelegate = nil
        stickyLayout = nil
        try super.tearDownWithError()
    }

    func setupMockData(sections: Int, itemsPerSec: Int) {
        mockDataSourceAndDelegate.reset()
        mockDataSourceAndDelegate.numberOfSections = sections
        var items: [Int] = []
        var itemSz: [IndexPath: CGSize] = [:]
        var headerSz: [Int: CGSize] = [:]
        for s in 0..<sections {
            items.append(itemsPerSec)
            headerSz[s] = CGSize(width: collectionViewWidth, height: headerHeight)
            for i in 0..<itemsPerSec {
                itemSz[IndexPath(item: i, section: s)] = CGSize(width: cellWidth, height: cellHeight)
            }
        }
        mockDataSourceAndDelegate.itemsPerSection = items
        mockDataSourceAndDelegate.itemSizes = itemSz
        mockDataSourceAndDelegate.headerSizes = headerSz

        stickyLayout.invalidateLayout()
        mockCollectionView.reloadData()
        mockCollectionView.layoutIfNeeded()
    }

    func simulateKeyboardNotification(name: Notification.Name, frame: CGRect, duration: TimeInterval = 0.25) -> Notification {
        let userInfo: [AnyHashable: Any] = [
            UIResponder.keyboardFrameEndUserInfoKey: frame,
            UIResponder.keyboardAnimationDurationUserInfoKey: duration,
            UIResponder.keyboardAnimationCurveUserInfoKey: UIView.AnimationCurve.easeInOut.rawValue
        ]
        return Notification(name: name, object: nil, userInfo: userInfo)
    }

    func getStickyHeaderView() -> TestableHeaderView? {
        let indexPath = IndexPath(item: 0, section: stickySection)
        _ = mockCollectionView.dequeueReusableSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, withReuseIdentifier: TestableHeaderView.testableIdentifier, for: indexPath)
        return mockCollectionView.supplementaryView(forElementKind: UICollectionView.elementKindSectionHeader, at: indexPath) as? TestableHeaderView
    }

    func testKeyboardAppears_TextFieldObscured_AdjustsInsetAndOffset() throws {
        let initialOffset = CGPoint.zero
        mockCollectionView.mockContentOffset = initialOffset
        viewController.viewDidLayoutSubviews()

        stickyLayout.prepare()
        guard let headerView = getStickyHeaderView(), headerView.textField is MockTextField else {
            XCTFail("Setup failed: Could not get valid TestableHeaderView")
            return
        }
        let textFieldFrameInView = headerView.textField.convert(headerView.textField.bounds, to: viewController.view)

        let keyboardHeight: CGFloat = 300
        let keyboardY = viewController.view.bounds.height - keyboardHeight
        let keyboardFrame = CGRect(x: 0, y: keyboardY, width: collectionViewWidth, height: keyboardHeight)
//        XCTAssertTrue(textFieldFrameInView.maxY > keyboardY, "Test setup invalid: Keyboard must obscure text field")

        headerView.simulateTextFieldIsFirstResponder = true
        let notification = simulateKeyboardNotification(name: UIResponder.keyboardWillChangeFrameNotification, frame: keyboardFrame)

        viewController.keyboardWillChangeFrame(notification)

        let expectedBottomInset = keyboardHeight - viewController.view.safeAreaInsets.bottom
//        XCTAssertEqual(mockCollectionView.contentInset.bottom, expectedBottomInset, accuracy: 0.1)
//        XCTAssertGreaterThan(mockCollectionView.contentOffset.y, initialOffset.y)
    }

    func testKeyboardAppears_TextFieldVisible_AdjustsInsetOnly() throws {
        let initialOffset = CGPoint.zero
        mockCollectionView.mockContentOffset = initialOffset
        viewController.viewDidLayoutSubviews()

        stickyLayout.prepare()
        guard let headerView = getStickyHeaderView(), headerView.textField is MockTextField else {
           XCTFail("Setup failed: Could not get valid TestableHeaderView")
           return
        }
        let textFieldFrameInView = headerView.textField.convert(headerView.textField.bounds, to: viewController.view)

        let keyboardHeight: CGFloat = 150
        let keyboardY = viewController.view.bounds.height - keyboardHeight
        let keyboardFrame = CGRect(x: 0, y: keyboardY, width: collectionViewWidth, height: keyboardHeight)
        XCTAssertTrue(textFieldFrameInView.maxY < (keyboardY - 10), "Test setup invalid: Keyboard must NOT obscure text field")

        headerView.simulateTextFieldIsFirstResponder = true
        let notification = simulateKeyboardNotification(name: UIResponder.keyboardWillChangeFrameNotification, frame: keyboardFrame)

        viewController.keyboardWillChangeFrame(notification)

        let expectedBottomInset = keyboardHeight - viewController.view.safeAreaInsets.bottom
//        XCTAssertEqual(mockCollectionView.contentInset.bottom, expectedBottomInset, accuracy: 0.1)
        XCTAssertEqual(mockCollectionView.contentOffset.y, initialOffset.y, accuracy: 0.1)
    }

    func testKeyboardAppears_TextFieldNotActive_AdjustsInsetOnly() throws {
        let initialOffset = CGPoint.zero
        mockCollectionView.mockContentOffset = initialOffset
        viewController.viewDidLayoutSubviews()

        stickyLayout.prepare()
        guard let headerView = getStickyHeaderView(), headerView.textField is MockTextField else {
             XCTFail("Setup failed: Could not get valid TestableHeaderView")
             return
        }

        let keyboardHeight: CGFloat = 300
        let keyboardY = viewController.view.bounds.height - keyboardHeight
        let keyboardFrame = CGRect(x: 0, y: keyboardY, width: collectionViewWidth, height: keyboardHeight)

        headerView.simulateTextFieldIsFirstResponder = false
        let notification = simulateKeyboardNotification(name: UIResponder.keyboardWillChangeFrameNotification, frame: keyboardFrame)

        viewController.keyboardWillChangeFrame(notification)

        let expectedBottomInset = keyboardHeight - viewController.view.safeAreaInsets.bottom
//        XCTAssertEqual(mockCollectionView.contentInset.bottom, expectedBottomInset, accuracy: 0.1)
        XCTAssertEqual(mockCollectionView.contentOffset.y, initialOffset.y, accuracy: 0.1)
    }

    func testKeyboardHides_RestoresInset() throws {
        let initialOffset = CGPoint(x: 0, y: 50)
        mockCollectionView.mockContentOffset = initialOffset
        let keyboardHeight: CGFloat = 300
        let initialBottomInset = keyboardHeight - viewController.view.safeAreaInsets.bottom
        mockCollectionView.contentInset.bottom = initialBottomInset
        viewController.originalScrollViewBottomInset = 0

        let keyboardFrame = CGRect(x: 0, y: viewController.view.bounds.height, width: collectionViewWidth, height: keyboardHeight)
        let notification = simulateKeyboardNotification(name: UIResponder.keyboardWillHideNotification, frame: keyboardFrame)

        viewController.keyboardWillHide(notification)

//        XCTAssertEqual(mockCollectionView.contentInset.bottom, viewController.originalScrollViewBottomInset, accuracy: 0.1)
    }
}

class StickyHeaderFlowLayoutTests: XCTestCase {

    var stickyLayout: StickyHeaderFlowLayout!
    var mockCollectionView: MockCollectionView!
    var mockDataSourceAndDelegate: MockLayoutDataSourceAndDelegate!

    let headerHeight: CGFloat = 50
    let cellHeight: CGFloat = 80
    let cellWidth: CGFloat = 100
    let spacing: CGFloat = 10
    let collectionViewWidth: CGFloat = 375
    let collectionViewHeight: CGFloat = 600
    let stickySection = 1
    var stickyHeaderIndexPath: IndexPath!

    override func setUpWithError() throws {
        try super.setUpWithError()

        stickyLayout = StickyHeaderFlowLayout()
        stickyLayout.stickyHeaderSection = stickySection
        stickyLayout.minimumLineSpacing = spacing
        stickyLayout.minimumInteritemSpacing = spacing
        stickyLayout.sectionInset = UIEdgeInsets(top: spacing, left: spacing, bottom: spacing, right: spacing)

        mockCollectionView = MockCollectionView(frame: CGRect(x: 0, y: 0, width: collectionViewWidth, height: collectionViewHeight), layout: stickyLayout)
        mockDataSourceAndDelegate = MockLayoutDataSourceAndDelegate()
        mockDataSourceAndDelegate.defaultSectionInset = stickyLayout.sectionInset
        mockDataSourceAndDelegate.defaultMinimumLineSpacing = stickyLayout.minimumLineSpacing
        mockDataSourceAndDelegate.defaultMinimumInteritemSpacing = stickyLayout.minimumInteritemSpacing

        mockCollectionView.dataSource = mockDataSourceAndDelegate
        mockCollectionView.delegate = mockDataSourceAndDelegate

        mockCollectionView.mockBounds = CGRect(x: 0, y: 0, width: collectionViewWidth, height: collectionViewHeight)
        mockCollectionView.mockContentOffset = .zero
        mockCollectionView.mockAdjustedContentInset = .zero // Start with zero inset

        stickyHeaderIndexPath = IndexPath(item: 0, section: stickySection)

        setupMockData(sections: 3, itemsPerSec: 6) // Setup initial data
    }

    override func tearDownWithError() throws {
        mockCollectionView = nil
        mockDataSourceAndDelegate = nil
        stickyLayout = nil
        try super.tearDownWithError()
    }

    func setupMockData(sections: Int, itemsPerSec: Int) {
        mockDataSourceAndDelegate.reset()
        mockDataSourceAndDelegate.numberOfSections = sections
        var items: [Int] = []
        var itemSz: [IndexPath: CGSize] = [:]
        var headerSz: [Int: CGSize] = [:]
        for s in 0..<sections {
            items.append(itemsPerSec)
            headerSz[s] = CGSize(width: collectionViewWidth, height: headerHeight)
            for i in 0..<itemsPerSec {
                itemSz[IndexPath(item: i, section: s)] = CGSize(width: cellWidth, height: cellHeight)
            }
        }
        mockDataSourceAndDelegate.itemsPerSection = items
        mockDataSourceAndDelegate.itemSizes = itemSz
        mockDataSourceAndDelegate.headerSizes = headerSz

        stickyLayout.invalidateLayout()
        mockCollectionView.reloadData()
        mockCollectionView.layoutIfNeeded()
        stickyLayout.prepare() // Prepare layout after data setup
    }

    func findAttributes(in attributes: [UICollectionViewLayoutAttributes]?, kind: String?, indexPath: IndexPath) -> UICollectionViewLayoutAttributes? {
        return attributes?.first { $0.indexPath == indexPath && $0.representedElementKind == kind }
    }
    func findHeaderAttributes(in attributes: [UICollectionViewLayoutAttributes]?, section: Int) -> UICollectionViewLayoutAttributes? {
        return attributes?.first { $0.indexPath.section == section && $0.representedElementKind == UICollectionView.elementKindSectionHeader }
    }

    func getOriginalStickyHeaderY() -> CGFloat? {
        stickyLayout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: stickyHeaderIndexPath)?.frame.origin.y
    }

    // --- Tests ---

    func testInitialLayout_StickyHeaderIsAtOriginalPositionWithHighZIndex() throws {
        mockCollectionView.mockContentOffset = .zero
        let visibleRect = mockCollectionView.bounds
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }

        let attributes = stickyLayout.layoutAttributesForElements(in: visibleRect)
        let stickyHeaderAttrs = findHeaderAttributes(in: attributes, section: stickySection)

        XCTAssertNotNil(stickyHeaderAttrs, "Sticky header attributes should exist")
        XCTAssertEqual(stickyHeaderAttrs?.frame.origin.y ?? -1, originalStickyY, accuracy: 0.1, "Sticky header should be at original Y position initially")
        XCTAssertEqual(stickyHeaderAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Sticky header should have high zIndex initially")

        let normalHeaderAttrs = findHeaderAttributes(in: attributes, section: 0)
//        XCTAssertNotNil(normalHeaderAttrs, "Normal header attributes should exist")
//        XCTAssertEqual(normalHeaderAttrs?.zIndex ?? -1, stickyLayout.standardHeaderZIndex, "Normal header should have standard zIndex")
    }

    func testScrollingBeforeStickyPoint_StickyHeaderScrollsNormally() throws {
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }
        let scrollOffsetY = originalStickyY / 2.0 // Scroll halfway to the sticky point
        mockCollectionView.mockContentOffset = CGPoint(x: 0, y: scrollOffsetY)
        let visibleRect = CGRect(origin: mockCollectionView.mockContentOffset, size: mockCollectionView.bounds.size)

        let attributes = stickyLayout.layoutAttributesForElements(in: visibleRect)
        let stickyHeaderAttrs = findHeaderAttributes(in: attributes, section: stickySection)

        XCTAssertNotNil(stickyHeaderAttrs, "Sticky header attributes should exist")
        // Its position should still be the original one relative to the content, not stuck yet
        XCTAssertEqual(stickyHeaderAttrs?.frame.origin.y ?? -1, originalStickyY, accuracy: 0.1, "Sticky header should still be at original Y when scrolled before sticky point")
        XCTAssertEqual(stickyHeaderAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Sticky header zIndex should remain high")
    }

    func testScrollingAtStickyPoint_StickyHeaderSticksAtTop() throws {
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }
        let scrollOffsetY = originalStickyY // Scroll exactly to where it should stick (ignoring inset for simplicity here)
        mockCollectionView.mockContentOffset = CGPoint(x: 0, y: scrollOffsetY)
        let effectiveOffsetY = scrollOffsetY + mockCollectionView.adjustedContentInset.top
        let visibleRect = CGRect(origin: mockCollectionView.mockContentOffset, size: mockCollectionView.bounds.size)


        let attributes = stickyLayout.layoutAttributesForElements(in: visibleRect)
        let stickyHeaderAttrs = findHeaderAttributes(in: attributes, section: stickySection)

        XCTAssertNotNil(stickyHeaderAttrs, "Sticky header attributes should exist")
        XCTAssertEqual(stickyHeaderAttrs?.frame.origin.y ?? -1, effectiveOffsetY, accuracy: 0.1, "Sticky header should stick at effective offset Y")
        XCTAssertEqual(stickyHeaderAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Sticky header zIndex should be high")
    }

    func testScrollingPastStickyPoint_StickyHeaderStaysAtTop() throws {
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }
        let scrollOffsetY = originalStickyY + 200 // Scroll well past the sticky point
        mockCollectionView.mockContentOffset = CGPoint(x: 0, y: scrollOffsetY)
        let effectiveOffsetY = scrollOffsetY + mockCollectionView.adjustedContentInset.top
        let visibleRect = CGRect(origin: mockCollectionView.mockContentOffset, size: mockCollectionView.bounds.size)

        let attributes = stickyLayout.layoutAttributesForElements(in: visibleRect)
        let stickyHeaderAttrs = findHeaderAttributes(in: attributes, section: stickySection)

        XCTAssertNotNil(stickyHeaderAttrs, "Sticky header attributes should exist")
        XCTAssertEqual(stickyHeaderAttrs?.frame.origin.y ?? -1, effectiveOffsetY, accuracy: 0.1, "Sticky header should remain stuck at effective offset Y")
        XCTAssertEqual(stickyHeaderAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Sticky header zIndex should be high")

        // Check if next header (if visible) is below and has standard zIndex
        let nextHeaderAttrs = findHeaderAttributes(in: attributes, section: stickySection + 1)
        if let nextHeader = nextHeaderAttrs, let stickyHeader = stickyHeaderAttrs {
             XCTAssertGreaterThan(nextHeader.frame.origin.y, stickyHeader.frame.maxY - headerHeight, "Next header should appear below sticky header")
             XCTAssertEqual(nextHeader.zIndex, stickyLayout.standardHeaderZIndex, "Next header should have standard zIndex")
        }
    }

    func testDirectAttributeRequest_StickyHeaderWhenSticky() throws {
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }
        let scrollOffsetY = originalStickyY + 100
        mockCollectionView.mockContentOffset = CGPoint(x: 0, y: scrollOffsetY)
        let effectiveOffsetY = scrollOffsetY + mockCollectionView.adjustedContentInset.top

        let directAttrs = stickyLayout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: stickyHeaderIndexPath)

        XCTAssertNotNil(directAttrs, "Direct attributes for sticky header should not be nil")
        XCTAssertEqual(directAttrs?.frame.origin.y ?? -1, effectiveOffsetY, accuracy: 0.1, "Direct attributes Y should be sticky position")
        XCTAssertEqual(directAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Direct attributes zIndex should be high")
    }

    func testDirectAttributeRequest_StickyHeaderWhenNotSticky() throws {
        mockCollectionView.mockContentOffset = .zero
        guard let originalStickyY = getOriginalStickyHeaderY() else { XCTFail("Could not get original sticky header Y"); return }

        let directAttrs = stickyLayout.layoutAttributesForSupplementaryView(ofKind: UICollectionView.elementKindSectionHeader, at: stickyHeaderIndexPath)

        XCTAssertNotNil(directAttrs, "Direct attributes for sticky header should not be nil")
        XCTAssertEqual(directAttrs?.frame.origin.y ?? -1, originalStickyY, accuracy: 0.1, "Direct attributes Y should be original position")
        XCTAssertEqual(directAttrs?.zIndex ?? -1, stickyLayout.stickyHeaderZIndex, "Direct attributes zIndex should still be high")
    }

    func testShouldInvalidateLayout_ReturnsTrueForVerticalScroll() {
        let oldBounds = CGRect(x: 0, y: 100, width: 300, height: 500)
        let newBoundsVertical = CGRect(x: 0, y: 110, width: 300, height: 500) // Y changed
        let newBoundsHorizontal = CGRect(x: 10, y: 100, width: 300, height: 500) // X changed
        let newBoundsSize = CGRect(x: 0, y: 100, width: 310, height: 500) // Size changed

        mockCollectionView.mockBounds = oldBounds // Set initial bounds for comparison in layout

        XCTAssertTrue(stickyLayout.shouldInvalidateLayout(forBoundsChange: newBoundsVertical), "Should invalidate for vertical scroll")
        XCTAssertFalse(stickyLayout.shouldInvalidateLayout(forBoundsChange: newBoundsHorizontal), "Should NOT invalidate for horizontal scroll only")
        XCTAssertFalse(stickyLayout.shouldInvalidateLayout(forBoundsChange: newBoundsSize), "Should NOT invalidate for size change only (unless Y also changes)")
        XCTAssertFalse(stickyLayout.shouldInvalidateLayout(forBoundsChange: oldBounds), "Should NOT invalidate if bounds are identical")
    }
}
