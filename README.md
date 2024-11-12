# magento2-module-gallery



To create a Magento 2 module that allows admin users to upload an image and select where they want to use it (e.g., gallery, thumbnail, etc.) within a backend grid, we’ll follow these steps:

Module Setup: Create the module file structure.
Database Schema: Add database columns to store image paths and roles.
Admin Grid: Create an admin grid with options for image roles.
Image Upload: Implement image upload functionality.
Save Image Roles: Allow admins to select roles for the uploaded image.
Below is the complete code for a custom module to achieve this.

Step 1: Module Setup
Create the folder structure:

```
app/code/YourVendor/ImageRoleSelector
├── Block
│   └── Adminhtml
│       └── ImageRole.php
├── Controller
│   └── Adminhtml
│       └── Entity
│           ├── Index.php
│           ├── Edit.php
│           ├── Save.php
│           └── Upload.php
├── Model
│   └── ResourceModel
│       └── Entity
│           ├── Collection.php
│           └── Entity.php
│       └── Entity.php
├── view
│   └── adminhtml
│       └── ui_component
│           └── image_role_grid.xml
└── registration.php
```
Step 2: Registration and Module Configuration
registration.php

```php
<?php

\Magento\Framework\Component\ComponentRegistrar::register(
    \Magento\Framework\Component\ComponentRegistrar::MODULE,
    'YourVendor_ImageRoleSelector',
    __DIR__
);
```

`etc/module.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="YourVendor_ImageRoleSelector" setup_version="1.0.0"/>
</config>
```
Step 3: Database Schema
Create a db_schema.xml file to define the table.

`etc/db_schema.xml`
```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/db_schema.xsd">
    <table name="yourvendor_image_role_selector" resource="default" engine="innodb" comment="Image Role Selector Table">
        <column xsi:type="int" name="entity_id" nullable="false" identity="true" unsigned="true" comment="Entity ID"/>
        <column xsi:type="varchar" name="image" nullable="true" length="255" comment="Image Path"/>
        <column xsi:type="varchar" name="role" nullable="true" length="255" comment="Image Role"/>
        <constraint xsi:type="primary" referenceId="PRIMARY">
            <column name="entity_id"/>
        </constraint>
    </table>
</schema>
```
Step 4: Admin Grid and Form
Create a UI component XML file for the admin grid and a form for uploading/selecting image roles.

`view/adminhtml/ui_component/image_role_grid.xml`
```xml
<listing xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Ui/etc/ui_component.xsd">
    <argument name="data" xsi:type="array">
        <item name="js_config" xsi:type="array">
            <item name="provider" xsi:type="string">image_role_grid.image_role_grid_data_source</item>
        </item>
    </argument>
    <dataSource name="image_role_grid_data_source">
        <argument name="dataProvider" xsi:type="configurableObject">
            <argument name="class" xsi:type="string">YourVendor\ImageRoleSelector\Model\ResourceModel\Entity\Collection</argument>
        </argument>
    </dataSource>
    <columns name="image_role_columns">
        <column name="image" class="Magento\Ui\Component\Listing\Columns\File">
            <settings>
                <label translate="true">Image</label>
            </settings>
        </column>
        <column name="role" class="Magento\Ui\Component\Listing\Columns\Select">
            <settings>
                <label translate="true">Role</label>
                <options>
                    <option name="base" xsi:type="array" xsi:type="string">Base Image</option>
                    <option name="small" xsi:type="array" xsi:type="string">Small Image</option>
                    <option name="thumbnail" xsi:type="array" xsi:type="string">Thumbnail</option>
                </options>
            </settings>
        </column>
    </columns>
</listing>
```

`Controller/Adminhtml/Entity/Upload.php`

```php
<?php

namespace YourVendor\ImageRoleSelector\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Framework\Controller\Result\JsonFactory;
use Magento\MediaStorage\Model\File\UploaderFactory;
use Magento\Framework\Filesystem;
use Magento\Framework\App\Filesystem\DirectoryList;

class Upload extends Action
{
    protected $resultJsonFactory;
    protected $uploaderFactory;
    protected $filesystem;

    public function __construct(
        Action\Context $context,
        JsonFactory $resultJsonFactory,
        UploaderFactory $uploaderFactory,
        Filesystem $filesystem
    ) {
        $this->resultJsonFactory = $resultJsonFactory;
        $this->uploaderFactory = $uploaderFactory;
        $this->filesystem = $filesystem;
        parent::__construct($context);
    }

    public function execute()
    {
        $result = $this->resultJsonFactory->create();
        try {
            $uploader = $this->uploaderFactory->create(['fileId' => 'image']);
            $uploader->setAllowedExtensions(['jpg', 'jpeg', 'png', 'gif']);
            $uploader->setAllowRenameFiles(true);
            $mediaDirectory = $this->filesystem->getDirectoryWrite(DirectoryList::MEDIA);
            $resultSave = $uploader->save($mediaDirectory->getAbsolutePath('custom/images'));
            if ($resultSave) {
                $result->setData([
                    'file' => 'custom/images/' . $resultSave['file'],
                    'url' => $this->getUrl('pub/media/custom/images/' . $resultSave['file']),
                ]);
            }
        } catch (\Exception $e) {
            $result->setData(['error' => $e->getMessage()]);
        }
        return $result;
    }
}
```
Step 6: Save Controller
In the Save controller, save the image and the selected role.

`Controller/Adminhtml/Entity/Save.php`

```php
<?php

namespace YourVendor\ImageRoleSelector\Controller\Adminhtml\Entity;

use Magento\Backend\App\Action;
use Magento\Framework\Exception\LocalizedException;

class Save extends Action
{
    public function execute()
    {
        $data = $this->getRequest()->getPostValue();

        if (isset($data['image']) && isset($data['role'])) {
            $model = $this->_objectManager->create('YourVendor\ImageRoleSelector\Model\Entity');
            $model->setData('image', $data['image']);
            $model->setData('role', $data['role']);
            try {
                $model->save();
                $this->messageManager->addSuccessMessage(__('Image and role have been saved.'));
            } catch (\Exception $e) {
                $this->messageManager->addErrorMessage($e->getMessage());
            }
        }
        return $this->_redirect('*/*/');
    }
}
```
This provides the foundation for a custom Magento 2 module that allows image upload with role selection in the backend grid. Additional work on the frontend could be done if needed for further functionality.
